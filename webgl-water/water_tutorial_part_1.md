# Rebuilding Evan Wallace’s WebGL Water Demo — WebGL Fundamentals Style

This tutorial reconstructs Evan Wallace’s famous **WebGL Water** demo step‑by‑step.

It intentionally follows the coding style of the original project:

- WebGL 1
- `lightgl.js`
- GLSL embedded in JS strings
- no modern JS modules

The tutorial is also written in the **WebGL Fundamentals teaching style**:

• Every working stage has complete code
• Changes between stages are shown as **code deltas**
• Each shader and JS block is explained line‑by‑line

The goal is to make the entire architecture understandable from scratch.

---

# Roadmap

We will build the demo in six stages.

| Part | Result |
|-----|------|
| 1 | GPU height‑field simulation you can poke |
| 2 | Water surface mesh displaced by the height field |
| 3 | Reflection, refraction, Fresnel water shading |
| 4 | Pool walls and sphere integrated into the scene |
| 5 | Caustics projection from the water surface |
| 6 | Sphere‑water interaction and the full app loop |

Each stage ends with something **fully functional**.

---

# Part 1 — A Height‑Field Simulation

At the end of Part 1 you will have:

• a floating‑point simulation texture
• a shader that adds ripple disturbances
• a shader that propagates waves
• a shader that recomputes surface normals
• a debug renderer to visualize the data

This is the entire **physics layer** of the demo.

No water rendering yet.

---

# The Key Simplification

The demo does **not simulate a volume of water**.

Instead it simulates a **height field**.

The water surface is described as

h(x, z, t)

Each texel stores four values:

| Channel | Meaning |
|-------|--------|
| R | height |
| G | vertical velocity |
| B | normal.x |
| A | normal.z |

Normals are stored partially because the y component can be reconstructed.

This single texture drives both the **simulation** and the **rendering** later.

---

# Step 1 — Creating the GL Context

First create a minimal application shell.

## index.html

```html
<html>
<head>
<script src="lightgl.js"></script>
<script src="water.js"></script>
<script src="main.js"></script>
</head>
<body></body>
</html>
```

We will implement `water.js` and `main.js` next.

---

# Step 2 — Creating the Water Simulation Object

Create a new file:

## water.js

```javascript
function Water() {

  var vertexShader = '\\
    varying vec2 coord;\\
    void main() {\\
      coord = gl_Vertex.xy * 0.5 + 0.5;\\
      gl_Position = vec4(gl_Vertex.xyz, 1.0);\\
    }\\
  ';

  this.plane = GL.Mesh.plane();

  if (!GL.Texture.canUseFloatingPointTextures()) {
    throw new Error("floating point textures required");
  }

  var filter = GL.Texture.canUseFloatingPointLinearFiltering() ? gl.LINEAR : gl.NEAREST;

  this.textureA = new GL.Texture(256,256,{ type:gl.FLOAT, filter:filter });
  this.textureB = new GL.Texture(256,256,{ type:gl.FLOAT, filter:filter });

}
```

---

# Understanding This Code

### Shared Vertex Shader

```
coord = gl_Vertex.xy * 0.5 + 0.5
```

This converts coordinates from

[-1 , 1]

into

[0 , 1]

which matches texture UV space.

Every simulation pass will use this shader.

---

### Fullscreen Plane

```
this.plane = GL.Mesh.plane()
```

Simulation is implemented by drawing a fullscreen quad.

Each fragment corresponds to **one texel of the simulation texture**.

The fragment shader therefore behaves like a GPU compute kernel.

---

### Floating‑Point Textures

Water heights need precision.

8‑bit textures would cause severe quantization artifacts.

We allocate **two textures**.

```
textureA
textureB
```

These implement the standard **ping‑pong technique**.

One texture is read from while the other is written to.

---

# Step 3 — Adding the Drop Shader

Now we create the disturbance pass.

Add this inside the Water constructor.

```javascript
this.dropShader = new GL.Shader(vertexShader,'\\
const float PI = 3.141592653589793;\\
uniform sampler2D texture;\\
uniform vec2 center;\\
uniform float radius;\\
uniform float strength;\\
varying vec2 coord;\\
void main(){\\

vec4 info = texture2D(texture,coord);\\

float drop = max(0.0,1.0-length(center*0.5+0.5-coord)/radius);\\

drop = 0.5 - cos(drop*PI)*0.5;\\

info.r += drop*strength;\\

gl_FragColor = info;\\
}\\
');
```

---

# Understanding the Drop Shader

### Reading the Previous State

```
vec4 info = texture2D(texture,coord)
```

Each texel stores the current simulation state.

---

### Distance From Drop Center

```
length(center*0.5+0.5-coord)
```

The center is specified in simulation space

[-1 , 1]

but texture coordinates use

[0 , 1]

so we convert.

---

### Radial Falloff

```
1.0 - distance/radius
```

Inside the radius → positive value

Outside → clamped to zero.

---

### Smooth Bump Shape

```
0.5 - cos(x*pi)*0.5
```

This transforms the radial falloff into a **smooth wave packet**.

Without this smoothing the simulation would produce harsh artifacts.

---

# Step 4 — Running the Drop Shader

Add a method below the constructor.

```javascript
Water.prototype.addDrop = function(x,y,radius,strength){

var this_ = this;

this.textureB.drawTo(function(){

this_.textureA.bind();

this_.dropShader.uniforms({
center:[x,y],
radius:radius,
strength:strength
}).draw(this_.plane);

});

this.textureB.swapWith(this.textureA);

}
```

---

# Understanding the Ping‑Pong Pattern

Every simulation pass follows this pattern:

```
read from textureA
write into textureB
swap textures
```

This avoids reading and writing the same texture simultaneously.

---

# Step 5 — Wave Propagation Shader

Add another shader inside the constructor.

```javascript
this.updateShader = new GL.Shader(vertexShader,'\\
uniform sampler2D texture;\\
uniform vec2 delta;\\
varying vec2 coord;\\
void main(){\\

vec4 info = texture2D(texture,coord);\\

vec2 dx = vec2(delta.x,0.0);\\
vec2 dy = vec2(0.0,delta.y);\\

float average = (\\
texture2D(texture,coord-dx).r +\\
texture2D(texture,coord+dx).r +\\
texture2D(texture,coord-dy).r +\\
texture2D(texture,coord+dy).r\\
)*0.25;\\

info.g += (average - info.r) * 2.0;\\

info.g *= 0.995;\\

info.r += info.g;\\

gl_FragColor = info;\\
}\\
');
```

---

# The Physics Behind This Shader

This implements a discrete wave equation.

Each texel behaves like a **mass connected by springs** to its neighbors.

Steps:

1 compute neighbor average

2 accelerate velocity toward that average

3 damp velocity

4 advance height

Velocity is necessary to produce **oscillation instead of diffusion**.

---

# Step 6 — Running the Simulation Step

Add another method.

```javascript
Water.prototype.stepSimulation = function(){

var this_ = this;

this.textureB.drawTo(function(){

this_.textureA.bind();

this_.updateShader.uniforms({

delta:[
1/this_.textureA.width,
1/this_.textureA.height
]

}).draw(this_.plane);

});

this.textureB.swapWith(this.textureA);

}
```

`delta` is the size of one texel in UV coordinates.

---

# Step 7 — Computing Surface Normals

Add the third shader.

```javascript
this.normalShader = new GL.Shader(vertexShader,'\\
uniform sampler2D texture;\\
uniform vec2 delta;\\
varying vec2 coord;\\
void main(){\\

vec4 info = texture2D(texture,coord);\\

vec3 dx = vec3(

delta.x,

texture2D(texture,vec2(coord.x+delta.x,coord.y)).r-info.r,

0.0

);

vec3 dy = vec3(

0.0,

texture2D(texture,vec2(coord.x,coord.y+delta.y)).r-info.r,

delta.y

);

info.ba = normalize(cross(dy,dx)).xz;

gl_FragColor = info;

}

');
```

---

# Understanding the Normal Calculation

We approximate surface derivatives:

```
dx = (1 , height difference , 0)

dy = (0 , height difference , 1)
```

Their cross product gives the surface normal.

Only the x and z components are stored.

The y component is reconstructed later.

---

# Step 8 — Updating Normals

```javascript
Water.prototype.updateNormals = function(){

var this_ = this;

this.textureB.drawTo(function(){

this_.textureA.bind();

this_.normalShader.uniforms({

delta:[
1/this_.textureA.width,
1/this_.textureA.height
]

}).draw(this_.plane);

});

this.textureB.swapWith(this.textureA);

}
```

---

# Step 9 — Minimal Application

Create main.js

```javascript
var gl = GL.create();

var water;

window.onload = function(){

water = new Water();

document.body.appendChild(gl.canvas);

gl.canvas.width = 600;

gl.canvas.height = 600;

gl.viewport(0,0,600,600);

water.addDrop(0,0,0.1,0.05);

animate();

}

function animate(){

water.stepSimulation();

water.stepSimulation();

water.updateNormals();

draw();

requestAnimationFrame(animate);

}
```

---

# Step 10 — Debug Renderer

We still need to see the simulation.

Add this to main.js

```javascript
var debugShader = new GL.Shader(

'\\
varying vec2 coord;\\
void main(){\\
coord = gl_Vertex.xy*0.5+0.5;\\
gl_Position = vec4(gl_Vertex.xyz,1.0);\\
}\\
',

'\\
uniform sampler2D texture;\\
varying vec2 coord;\\
void main(){\\
vec4 info = texture2D(texture,coord);\\
gl_FragColor = vec4(vec3(info.r*0.5+0.5),1.0);\\
}\\
'

);

var debugPlane = GL.Mesh.plane();

function draw(){

gl.clear(gl.COLOR_BUFFER_BIT);

water.textureA.bind(0);


debugShader.uniforms({texture:0}).draw(debugPlane);

}
```

---

# Simulation Data Layout

Understanding the data layout inside the water texture is critical before continuing.

```
Water Simulation Texture (RGBA per texel)

+-----+-----+-----+-----+
|  R  |  G  |  B  |  A  |
+-----+-----+-----+-----+

R = surface height
G = vertical velocity
B = normal.x
A = normal.z
```

Why store these together?

• The simulation updates **height and velocity**.
• Rendering later needs **surface normals**.
• Packing them into one texture keeps the GPU passes simple and reduces texture lookups.

Notice that **normal.y is not stored**.

Because normals are unit length we can reconstruct it later:

```
normal.y = sqrt(1 - normal.x² - normal.z²)
```

This saves one texture channel.

---

# Ping‑Pong Simulation

GPU simulations cannot safely read and write the same texture in a single pass.

Instead we alternate between two textures.

```
Frame N

textureA  (current state)
     │
     ▼
 draw fullscreen quad
 (fragment shader runs once per texel)
     │
     ▼
textureB  (next state)

swap(textureA, textureB)
```

After the swap the new state becomes the input for the next step.

Every simulation pass in this project follows this exact pattern.

---

# Simulation Pass Pipeline

Each animation frame performs several GPU passes.

```
User interaction
(addDrop)

        │
        ▼

stepSimulation()

        │
        ▼

stepSimulation()

        │
        ▼

updateNormals()

        │
        ▼

render
```

Two simulation steps per frame make the waves propagate faster without increasing the render frame rate.

Normals are recomputed **after** the simulation because they depend on the latest heights.

---

# Result of Part 1

You should now see:

• expanding ripples
• smooth wave propagation
• fully GPU‑based simulation

But the water is still just a grayscale texture.

In **Part 2** we will turn this data into a real **3D water surface mesh**.

That step will introduce the renderer and vertex displacement.
