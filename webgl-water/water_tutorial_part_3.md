# WebGL Water Tutorial — Part 3: Reflection, Refraction, and Fresnel

In Parts 1–2 we built the water simulation and rendered the surface as displaced geometry.

In this part we make the surface behave **optically like water**.

At the end of Part 3 the surface will:

• reflect the sky
• refract rays into the pool
• change reflectivity depending on view angle (Fresnel)
• render differently from above and below the surface

This is the stage where the demo begins to look like real water.

---

# Part 3 Architecture

Nothing changes in the simulation layer. Only the renderer becomes more sophisticated.

```
water simulation (unchanged)
        │
        ▼
water texture (height + velocity + normals)
        │
        ▼
water surface shader
        │
   reflection + refraction
        │
        ▼
final color
```

The shader now treats the surface as the **boundary between two media**.

---

# Reflection vs Refraction

When a viewing ray hits water it splits into two rays.

```
              reflected ray
                   ↗
                  /
 camera ray ───► surface
                  \
                   ↘
               refracted ray
```

Reflection stays in air.

Refraction bends into the water.

Both rays must be evaluated.

---

# Fresnel Effect

The ratio of reflection to refraction depends on angle.

```
looking straight down
   mostly refraction

looking across surface
   mostly reflection
```

This is the **Fresnel effect** and it is one of the biggest visual cues for water.

---

# Code Changes Overview

Files modified in this part:

```
renderer.js
main.js
```

Unchanged:

```
water.js
```

New resources:

```
cubemap sky texture
```

---

# Step 1 — Add the Cubemap

Water reflects the environment.

So we add a cubemap representing the sky.

### main.js

Add a global:

```javascript
var cubemap;
```

Initialize it in `window.onload`:

```javascript
cubemap = new Cubemap({
  xneg: document.getElementById('xneg'),
  xpos: document.getElementById('xpos'),
  yneg: document.getElementById('yneg'),
  ypos: document.getElementById('ypos'),
  zneg: document.getElementById('zneg'),
  zpos: document.getElementById('zpos')
});
```

These images form the six faces of the sky cube.

---

# Step 2 — Replace the Simple Shader

In Part 2 the fragment shader was just Lambert lighting.

```
vec3 color = vec3(0.2,0.6,1.0) * diffuse;
```

Now we replace it with optical shading.

To avoid repeating code we first define shared helpers.

---

# Shared Shader Constants

```
const float IOR_AIR = 1.0;
const float IOR_WATER = 1.333;
```

These are **indices of refraction**.

They control how rays bend when crossing the interface.

When computing refraction we pass the ratio:

```
eta = n1 / n2
```

Examples:

```
air → water : IOR_AIR / IOR_WATER
water → air : IOR_WATER / IOR_AIR
```

---

# Step 3 — Vertex Shader (Full Code)

The vertex shader remains mostly the same.

```glsl
uniform sampler2D water;
varying vec3 position;

void main() {

  vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);

  position = gl_Vertex.xzy;
  position.y += info.r;

  gl_Position = gl_ModelViewProjectionMatrix * vec4(position,1.0);

}
```

Line-by-line:

```
texture2D(water,...)
```
reads the simulation texture.

```
info.r
```
is the water height.

The mesh vertex is displaced vertically.

---

# Mapping Diagram

```
water texture (simulation)

   UV
    │
    ▼
mesh vertex

(x,z) → sample height
```

Each vertex samples the same water simulation texture.

---

# Step 4 — Fragment Shader Normal Reconstruction

The simulation texture stores only two components of the normal.

```
B = normal.x
A = normal.z
```

The fragment shader reconstructs the missing Y component.

```glsl
vec2 coord = position.xz * 0.5 + 0.5;
vec4 info = texture2D(water, coord);

vec3 normal = vec3(
  info.b,
  sqrt(1.0 - dot(info.ba, info.ba)),
  info.a
);
```

Because the vector is unit length:

```
x² + y² + z² = 1
```

so

```
y = sqrt(1 − x² − z²)
```

---

# Step 5 — Reflection

Reflection uses GLSL's built-in function.

```glsl
vec3 reflectedRay = reflect(incomingRay, normal);
```

The environment color comes from the cubemap.

```glsl
vec3 reflectedColor = textureCube(sky, reflectedRay).rgb;
```

This produces the sky reflections.

---

# Step 6 — Refraction

Refraction bends the ray entering water.

```glsl
vec3 refractedRay = refract(
  incomingRay,
  normal,
  IOR_AIR / IOR_WATER
);
```

This ray travels downward into the pool.

The shader traces it against the pool walls.

---

# Ray Intersection Diagram

```
view ray hits water
        │
        ├── reflected → sky
        │
        └── refracted → pool interior
```

The shader computes both and blends them.

---

# Step 7 — Fresnel Approximation

Real Fresnel equations are expensive.

The demo uses a cheap approximation.

```glsl
float fresnel = mix(
  0.25,
  1.0,
  pow(1.0 - dot(normal, -incomingRay), 3.0)
);
```

Explanation:

```
dot(normal,view)
```
measures viewing angle.

When the angle becomes grazing the Fresnel term approaches 1.

This increases reflection.

---

# Step 8 — Combine Reflection and Refraction

The final color is a blend.

```glsl
vec3 color = mix(
  refractedColor,
  reflectedColor,
  fresnel
);
```

Meaning:

```
fresnel = 0 → mostly refraction
fresnel = 1 → mostly reflection
```

---

# Step 9 — Two‑Pass Water Rendering

Water behaves differently above and below the surface.

Instead of branching inside the shader we render twice.

```
Pass 1
  render back faces
  underwater shader

Pass 2
  render front faces
  above-water shader
```

Implementation:

```javascript
for (var i=0;i<2;i++) {

  gl.cullFace(i ? gl.BACK : gl.FRONT);

  this.waterShaders[i].draw(this.waterMesh);

}
```

Each shader uses a different refraction ratio.

---

# Rendering Function

Full renderer method.

```javascript
Renderer.prototype.renderWater = function(water, sky) {

  var tracer = new GL.Raytracer();

  water.textureA.bind(0);
  sky.bind(1);

  gl.enable(gl.CULL_FACE);

  for (var i=0;i<2;i++) {

    gl.cullFace(i ? gl.BACK : gl.FRONT);

    this.waterShaders[i].uniforms({
      water:0,
      sky:1,
      eye: tracer.eye
    }).draw(this.waterMesh);

  }

  gl.disable(gl.CULL_FACE);

}
```

The `Raytracer` helper provides the camera position used to compute the incoming view ray.

---

# Result of Part 3

At this stage you should see:

• reflections of the sky
• refraction through the water surface
• stronger reflections at grazing angles
• convincing optical water behavior

Even without the pool geometry, the surface already looks dramatically more realistic.

---

# What Comes Next

In **Part 4** we add real scene geometry:

• pool walls and floor
• the sphere inside the water

Those objects will interact with the reflection and refraction rays implemented here.

That is where the demo becomes a full scene.

