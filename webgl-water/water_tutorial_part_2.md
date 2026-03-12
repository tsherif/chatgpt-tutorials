# Part 2 — Turning the Height Field into Geometry

At the end of Part 2 you will have:

• a `Renderer` object
• a dense plane mesh for the water surface
• a vertex shader that samples the simulation texture
• displacement of the mesh by the stored water height
• reconstruction of the surface normal
• simple directional lighting

This is the first stage where the simulation data becomes actual 3D geometry.

The water will still be simple-looking. No reflection or refraction yet.

But it will already be the correct **shape**.

---

# What Changes in Part 2

Part 1 only displayed the simulation texture directly.

In Part 2 we will:

1. keep `water.js` exactly the same
2. replace the debug renderer in `main.js`
3. add a new file, `renderer.js`
4. render a dense mesh instead of a fullscreen debug plane

So the simulation stays untouched.

Only the **presentation of the data** changes.

---

# Code Delta Overview

## New file

```text
renderer.js
```

## Changed file

```text
main.js
```

## Unchanged file

```text
water.js
```

That is an important milestone.

It shows that the simulation and rendering are already nicely separated.

---

# Step 1 — Update the HTML File

We now need to load `renderer.js`.

## Delta: index.html

### Part 1

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

### Part 2

```html
<html>
<head>
<script src="lightgl.js"></script>
<script src="water.js"></script>
<script src="renderer.js"></script>
<script src="main.js"></script>
</head>
<body></body>
</html>
```

### What changed

We inserted:

```html
<script src="renderer.js"></script>
```

This new file will contain the geometry and shading code.

---

# Step 2 — Create the Renderer Object

Create a new file:

## renderer.js

```javascript
function Renderer() {

  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  this.waterShader = new GL.Shader('\
    uniform sampler2D water;\
    varying vec3 position;\
    varying vec3 normal;\
    void main() {\
      vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);\
      position = gl_Vertex.xzy;\
      position.y += info.r;\
      normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);\
      gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);\
    }\
  ', '\
    uniform vec3 light;\
    varying vec3 position;\
    varying vec3 normal;\
    void main() {\
      float diffuse = max(0.0, dot(normalize(normal), normalize(light)));\
      vec3 color = vec3(0.2, 0.6, 1.0) * (0.2 + 0.8 * diffuse);\
      gl_FragColor = vec4(color, 1.0);\
    }\
  ');

}
```

---

# Understanding the Water Mesh

```javascript
this.waterMesh = GL.Mesh.plane({ detail: 200 });
```

This creates a subdivided grid.

Why do we need that?

Because vertex displacement can only move **existing vertices**.

If the mesh had only four corners, the water would remain almost flat.

A dense grid gives the height field enough geometric resolution to become visibly wavy.

---

# Why `detail: 200`?

The simulation texture is 256×256.

The rendered mesh does not need to match that exactly.

It just needs enough vertices to:

• follow the displaced shape smoothly
• avoid obvious faceting
• remain reasonably fast

A 200×200-style subdivision is a good compromise.

---

# Step 3 — Understanding the Vertex Shader

Here is the vertex shader again by itself.

```glsl
uniform sampler2D water;
varying vec3 position;
varying vec3 normal;

void main() {
  vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);
  position = gl_Vertex.xzy;
  position.y += info.r;
  normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
  gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);
}
```

Now let’s go through it line by line.

---

## Sampling the Water Texture

```glsl
vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);
```

`gl_Vertex.xy` is the plane’s position in the range `[-1, 1]`.

The water texture uses UV coordinates in `[0, 1]`.

So just like in Part 1, we remap by multiplying by 0.5 and adding 0.5.

That means each water-mesh vertex samples the simulation state at its own horizontal location.

---

## Reinterpreting the Plane as an XZ Surface

```glsl
position = gl_Vertex.xzy;
```

The plane mesh is originally arranged in the XY plane.

But the water surface should lie in the XZ plane, with Y pointing upward.

So we swap axes.

That turns:

```text
(x, y, z)
```

into:

```text
(x, z, y)
```

Since the original plane has `z = 0`, this effectively creates a horizontal surface.

---

## Applying Height Displacement

```glsl
position.y += info.r;
```

This is the key step.

The red channel stores water height.

So every vertex moves upward by that amount.

This turns the flat grid into a real wave surface.

---

## Reconstructing the Normal

```glsl
normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
```

In Part 1 we stored only:

• `normal.x` in `B`
• `normal.z` in `A`

Because the normal is unit length, the missing y component is:

```glsl
sqrt(1.0 - x*x - z*z)
```

This is why we packed only two components into the simulation texture.

---

## Projecting the Vertex

```glsl
gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);
```

Now that the vertex has been displaced into world/model space, we project it normally.

---

# Step 4 — Understanding the Fragment Shader

Here is the fragment shader again.

```glsl
uniform vec3 light;
varying vec3 position;
varying vec3 normal;

void main() {
  float diffuse = max(0.0, dot(normalize(normal), normalize(light)));
  vec3 color = vec3(0.2, 0.6, 1.0) * (0.2 + 0.8 * diffuse);
  gl_FragColor = vec4(color, 1.0);
}
```

This is intentionally simple.

We are not trying to make the water realistic yet.

We just want to confirm that:

• the shape is correct
• the normals are correct
• the mesh is actually being displaced

---

## Diffuse Lighting

```glsl
float diffuse = max(0.0, dot(normalize(normal), normalize(light)));
```

This is standard Lambert lighting.

If the normal points toward the light, the surface is bright.

If it points away, it gets darker.

Because the normals come from the simulation texture, the lighting immediately reveals the wave structure.

---

## Base Color

```glsl
vec3 color = vec3(0.2, 0.6, 1.0) * (0.2 + 0.8 * diffuse);
```

This gives a simple blue tint.

We keep some ambient contribution:

```glsl
0.2
```

so the surface never goes fully black.

---

# Step 5 — Add a Render Method

Now add a method below the constructor in `renderer.js`.

```javascript
Renderer.prototype.renderWater = function(water) {

  water.textureA.bind(0);

  this.waterShader.uniforms({
    water: 0,
    light: [2, 2, -1]
  }).draw(this.waterMesh);

}
```

---

# Understanding `renderWater`

First we bind the latest simulation texture:

```javascript
water.textureA.bind(0)
```

Then we pass uniforms:

• `water: 0` tells the shader to sample texture unit 0
• `light: [2, 2, -1]` gives a simple fixed light direction

Then we draw the mesh.

This is the moment where the simulation becomes visible as geometry.

---

# Step 6 — Replace the Debug App with a Camera

Now we update `main.js`.

## Delta: main.js

### Part 1

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

var debugShader = new GL.Shader(

'\
varying vec2 coord;\
void main(){\
coord = gl_Vertex.xy*0.5+0.5;\
gl_Position = vec4(gl_Vertex.xyz,1.0);\
}\
',

'\
uniform sampler2D texture;\
varying vec2 coord;\
void main(){\
vec4 info = texture2D(texture,coord);\
gl_FragColor = vec4(vec3(info.r*0.5+0.5),1.0);\
}\
'

);

var debugPlane = GL.Mesh.plane();

function draw(){

gl.clear(gl.COLOR_BUFFER_BIT);

water.textureA.bind(0);


debugShader.uniforms({texture:0}).draw(debugPlane);

}
```

### Part 2

```javascript
var gl = GL.create();

var water;
var renderer;

var angleX = -25;
var angleY = -200.5;

window.onload = function() {

  water = new Water();
  renderer = new Renderer();

  document.body.appendChild(gl.canvas);
  gl.clearColor(0, 0, 0, 1);

  function onresize() {
    gl.canvas.width = innerWidth;
    gl.canvas.height = innerHeight;
    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

    gl.matrixMode(gl.PROJECTION);
    gl.loadIdentity();
    gl.perspective(45, gl.canvas.width / gl.canvas.height, 0.01, 100);

    gl.matrixMode(gl.MODELVIEW);
  }

  onresize();
  window.onresize = onresize;

  water.addDrop(0, 0, 0.08, 0.03);

  requestAnimationFrame(animate);

};

function animate() {

  water.stepSimulation();
  water.stepSimulation();
  water.updateNormals();

  draw();

  requestAnimationFrame(animate);
}

function draw() {

  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  gl.loadIdentity();

  gl.translate(0, 0, -4);
  gl.rotate(-angleX, 1, 0, 0);
  gl.rotate(-angleY, 0, 1, 0);
  gl.translate(0, 0.5, 0);

  gl.enable(gl.DEPTH_TEST);

  renderer.renderWater(water);

}
```

---

# What Changed in `main.js`

Several important things changed.

---

## We Created the Renderer

```javascript
renderer = new Renderer();
```

Part 1 had no separate rendering object.

Now rendering is split out into its own file and object.

---

## We Removed the Debug Shader

In Part 1 the simulation was shown directly as grayscale texture values.

That entire debug path is gone.

We no longer draw a fullscreen plane for inspection.

We now render actual water geometry.

---

## We Added a Projection Matrix

```javascript
gl.matrixMode(gl.PROJECTION);
gl.loadIdentity();
gl.perspective(45, gl.canvas.width / gl.canvas.height, 0.01, 100);
```

Part 1 did not need a camera because it was just drawing a fullscreen quad.

Part 2 draws a real 3D mesh, so we need perspective projection.

---

## We Added Camera Transforms

```javascript
gl.translate(0, 0, -4);
gl.rotate(-angleX, 1, 0, 0);
gl.rotate(-angleY, 0, 1, 0);
gl.translate(0, 0.5, 0);
```

This positions the camera to look down at the water surface from an angle.

For now these values are static.

Later, when we build the full application, they will also connect to user interaction.

---

## We Enabled Depth Testing

```javascript
gl.enable(gl.DEPTH_TEST);
```

Once we render 3D geometry, depth testing is required.

---

# Full Part 2 Files

Here is the complete code for the files used at the end of Part 2.

---

## index.html

```html
<html>
<head>
<script src="lightgl.js"></script>
<script src="water.js"></script>
<script src="renderer.js"></script>
<script src="main.js"></script>
</head>
<body></body>
</html>
```

---

## water.js

```javascript
function Water() {

  var vertexShader = '\
    varying vec2 coord;\
    void main() {\
      coord = gl_Vertex.xy * 0.5 + 0.5;\
      gl_Position = vec4(gl_Vertex.xyz, 1.0);\
    }\
  ';

  this.plane = GL.Mesh.plane();

  if (!GL.Texture.canUseFloatingPointTextures()) {
    throw new Error("floating point textures required");
  }

  var filter = GL.Texture.canUseFloatingPointLinearFiltering() ? gl.LINEAR : gl.NEAREST;

  this.textureA = new GL.Texture(256,256,{ type:gl.FLOAT, filter:filter });
  this.textureB = new GL.Texture(256,256,{ type:gl.FLOAT, filter:filter });

  this.dropShader = new GL.Shader(vertexShader,'\
const float PI = 3.141592653589793;\
uniform sampler2D texture;\
uniform vec2 center;\
uniform float radius;\
uniform float strength;\
varying vec2 coord;\
void main(){\
vec4 info = texture2D(texture,coord);\
float drop = max(0.0,1.0-length(center*0.5+0.5-coord)/radius);\
drop = 0.5 - cos(drop*PI)*0.5;\
info.r += drop*strength;\
gl_FragColor = info;\
}\
');

  this.updateShader = new GL.Shader(vertexShader,'\
uniform sampler2D texture;\
uniform vec2 delta;\
varying vec2 coord;\
void main(){\
vec4 info = texture2D(texture,coord);\
vec2 dx = vec2(delta.x,0.0);\
vec2 dy = vec2(0.0,delta.y);\
float average = (\
texture2D(texture,coord-dx).r +\
texture2D(texture,coord+dx).r +\
texture2D(texture,coord-dy).r +\
texture2D(texture,coord+dy).r\
)*0.25;\
info.g += (average - info.r) * 2.0;\
info.g *= 0.995;\
info.r += info.g;\
gl_FragColor = info;\
}\
');

  this.normalShader = new GL.Shader(vertexShader,'\
uniform sampler2D texture;\
uniform vec2 delta;\
varying vec2 coord;\
void main(){\
vec4 info = texture2D(texture,coord);\
vec3 dx = vec3(delta.x, texture2D(texture,vec2(coord.x+delta.x,coord.y)).r-info.r, 0.0);\
vec3 dy = vec3(0.0, texture2D(texture,vec2(coord.x,coord.y+delta.y)).r-info.r, delta.y);\
info.ba = normalize(cross(dy,dx)).xz;\
gl_FragColor = info;\
}\
');
}

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
};

Water.prototype.stepSimulation = function(){
  var this_ = this;
  this.textureB.drawTo(function(){
    this_.textureA.bind();
    this_.updateShader.uniforms({
      delta:[1/this_.textureA.width,1/this_.textureA.height]
    }).draw(this_.plane);
  });
  this.textureB.swapWith(this.textureA);
};

Water.prototype.updateNormals = function(){
  var this_ = this;
  this.textureB.drawTo(function(){
    this_.textureA.bind();
    this_.normalShader.uniforms({
      delta:[1/this_.textureA.width,1/this_.textureA.height]
    }).draw(this_.plane);
  });
  this.textureB.swapWith(this.textureA);
};
```

---

## renderer.js

```javascript
function Renderer() {

  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  this.waterShader = new GL.Shader('\
    uniform sampler2D water;\
    varying vec3 position;\
    varying vec3 normal;\
    void main() {\
      vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);\
      position = gl_Vertex.xzy;\
      position.y += info.r;\
      normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);\
      gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);\
    }\
  ', '\
    uniform vec3 light;\
    varying vec3 position;\
    varying vec3 normal;\
    void main() {\
      float diffuse = max(0.0, dot(normalize(normal), normalize(light)));\
      vec3 color = vec3(0.2, 0.6, 1.0) * (0.2 + 0.8 * diffuse);\
      gl_FragColor = vec4(color, 1.0);\
    }\
  ');

}

Renderer.prototype.renderWater = function(water) {
  water.textureA.bind(0);
  this.waterShader.uniforms({
    water: 0,
    light: [2, 2, -1]
  }).draw(this.waterMesh);
};
```

---

## main.js

```javascript
var gl = GL.create();

var water;
var renderer;

var angleX = -25;
var angleY = -200.5;

window.onload = function() {

  water = new Water();
  renderer = new Renderer();

  document.body.appendChild(gl.canvas);
  gl.clearColor(0, 0, 0, 1);

  function onresize() {
    gl.canvas.width = innerWidth;
    gl.canvas.height = innerHeight;
    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

    gl.matrixMode(gl.PROJECTION);
    gl.loadIdentity();
    gl.perspective(45, gl.canvas.width / gl.canvas.height, 0.01, 100);

    gl.matrixMode(gl.MODELVIEW);
  }

  onresize();
  window.onresize = onresize;

  water.addDrop(0, 0, 0.08, 0.03);

  requestAnimationFrame(animate);

};

function animate() {

  water.stepSimulation();
  water.stepSimulation();
  water.updateNormals();

  draw();

  requestAnimationFrame(animate);
}

function draw() {

  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  gl.loadIdentity();

  gl.translate(0, 0, -4);
  gl.rotate(-angleX, 1, 0, 0);
  gl.rotate(-angleY, 0, 1, 0);
  gl.translate(0, 0.5, 0);

  gl.enable(gl.DEPTH_TEST);

  renderer.renderWater(water);

}
```

---

# Geometry Mapping Diagram

This is the central idea of Part 2.

```
Simulation Texture
(height/velocity/normal)

      sample at
   vertex x,z location

        │
        ▼

flat subdivided plane
in object space

        │
        ▼

vertex shader adds info.r
to position.y

        │
        ▼

rippling 3D water mesh
```

The simulation still lives in texture space.

The renderer samples that texture and turns it into geometry.

---

# Coordinate Mapping Diagram

The plane mesh starts with coordinates in the XY plane.

```
original plane vertex
(gl_Vertex)

(x, y, 0)
```

The shader reinterprets it as water-surface coordinates:

```
position = gl_Vertex.xzy

(x, 0, y)
```

Then it adds the simulated height:

```
position.y += info.r

(x, height, z)
```

So the final water geometry lives in normal 3D world-style coordinates.

---

# Why Part 2 Matters

This stage establishes a very important architectural boundary.

`water.js` is purely about **simulation**.

`renderer.js` is purely about **turning simulation data into visible geometry**.

That separation is why the later stages are manageable.

We can now keep improving the visual appearance of the water without rewriting the underlying physics.

---

# Result of Part 2

At the end of this stage you should see:

• a blue water surface
• real wave displacement
• directional lighting that reveals the normal field
• the first true 3D representation of the simulated water

It still will not look like the final demo.

What is missing is the optical behavior that makes water look like water.

In **Part 3** we will add:

• reflection
• refraction
• Fresnel blending
• above-water and underwater shading paths

That is the stage where the surface starts to look convincing.

