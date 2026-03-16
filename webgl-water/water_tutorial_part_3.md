# WebGL Water Tutorial — Part 3: Reflection, Refraction, Fresnel, and Two-Sided Water

In Parts 1–2 we built:

- a GPU height-field simulation
- a displaced water mesh
- simple lighting from reconstructed normals

That gave us the **shape** of the water.

In this part we make the surface behave like an **optical interface**.

At the end of Part 3 the water will:

- reflect the sky
- refract rays into the pool
- change reflectivity with view angle
- render differently from above and below the surface

This is the point where the demo starts to look like water instead of blue wavy plastic.

---

# What Actually Changes in Part 3

The simulation still stays exactly the same.

`water.js` is unchanged.

What changes is the renderer:

- we add a sky cubemap
- we replace the single Lambert shader with a pair of water shaders
- we add shared GLSL helper functions
- we render the water in two passes:
  - once for back faces
  - once for front faces

That overall organization matches the reference demo’s renderer structure, where the heavy lifting is in `renderer.js` and the simulation remains separate in `water.js`.

---

# Why Two Water Shaders Exist

A water surface is the boundary between two media:

- air above
- water below

The optical behavior depends on which side of the interface the ray is coming from.

That changes:

- the normal direction we want to use
- the refraction ratio `eta`
- the color tint we apply

So instead of one shader full of branches, we build **two closely related fragment shaders**:

- shader 0: above-water shading
- shader 1: underwater shading

---

# Why Back Faces vs Front Faces Matter

The mesh is only one polygon sheet, but visually it has two sides.

For the water surface, rendering both sides lets us shade the interface differently depending on whether we are seeing:

- the top of the water
- the underside of the water

We do this with face culling:

- cull front faces for one pass
- cull back faces for the other pass

A useful mental model is:

```text
Pass 1: draw the underside of the sheet
Pass 2: draw the top side of the sheet
```

---

# Where the Helper Functions Fit

The helper functions are not separate shaders.

They are a **shared GLSL source string** that gets prepended to each water fragment shader.

That shared block contains things like:

- indices of refraction
- scene intersection helpers
- helper functions that answer “what color does this ray see?”

So the fragment shader can stay conceptually simple:

1. compute the normal
2. compute reflected and refracted rays
3. ask helper functions for the reflected and refracted colors
4. blend them with Fresnel

---

# Part 3 Architecture

```text
water simulation texture
(height, velocity, normal.x, normal.z)
           │
           ▼
    water vertex shader
(displace mesh using height)
           │
           ▼
  water fragment shader
(reconstruct normal)
           │
     ┌─────┴─────┐
     ▼           ▼
 reflection   refraction
     │           │
     └─────┬─────┘
           ▼
        Fresnel mix
           │
           ▼
      final water color
```

---

# Code Delta Overview

## New or changed resources in this part

### `index.html`
- add `cubemap.js`
- add hidden `<img>` elements for the cubemap faces

### `main.js`
- add a global `cubemap`
- initialize the cubemap in `window.onload`
- pass the cubemap into `renderer.renderWater()`

### `renderer.js`
- replace the single `waterShader` with `waterShaders[0]` and `waterShaders[1]`
- add a shared GLSL helper block
- update `renderWater()` to render in two passes with face culling

### `water.js`
- unchanged

---

# Step 1 — Delta: `index.html`

## Part 2

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

## Part 3

```html
<!DOCTYPE html>
<html>
<head>
  <script src="lightgl.js"></script>
  <script src="cubemap.js"></script>
  <script src="water.js"></script>
  <script src="renderer.js"></script>
  <script src="main.js"></script>
  <style>
    html, body { margin: 0; height: 100%; overflow: hidden; background: black; }
    canvas { display: block; width: 100%; height: 100%; }
    img { display: none; }
  </style>
</head>
<body>
  <img id="xneg" src="xneg.jpg">
  <img id="xpos" src="xpos.jpg">
  <img id="ypos" src="ypos.jpg">
  <img id="zneg" src="zneg.jpg">
  <img id="zpos" src="zpos.jpg">
</body>
</html>
```

## What changed

We added:

```html
<script src="cubemap.js"></script>
```

and hidden image elements for the sky cubemap faces.

We also added minimal CSS so the canvas fills the window cleanly.

---

# Step 2 — Delta: `main.js`

## Part 2

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

  gl.disable(gl.DEPTH_TEST);

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

## Part 3

```javascript
var gl = GL.create();

var water;
var renderer;
var cubemap;

var angleX = -25;
var angleY = -200.5;

window.onload = function() {

  water = new Water();
  renderer = new Renderer();

  cubemap = new Cubemap({
    xneg: document.getElementById('xneg'),
    xpos: document.getElementById('xpos'),
    yneg: document.getElementById('ypos'),
    ypos: document.getElementById('ypos'),
    zneg: document.getElementById('zneg'),
    zpos: document.getElementById('zpos')
  });

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

  gl.disable(gl.DEPTH_TEST);

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

  renderer.renderWater(water, cubemap);

}
```

## What changed

We added a new global:

```javascript
var cubemap;
```

and initialized it in `window.onload`.

Then we changed:

```javascript
renderer.renderWater(water);
```

into:

```javascript
renderer.renderWater(water, cubemap);
```

because the water shader now needs an environment map for reflections.

---

# Step 3 — Delta: `renderer.js` Constructor Overview

## Part 2 structure

```javascript
function Renderer() {

  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  this.waterShader = new GL.Shader('...vertex...', '...fragment...');

}
```

## Part 3 structure

```javascript
function Renderer() {

  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  var helperFunctions = '...shared GLSL...';
  var vertexShader = '...water displacement vertex shader...';

  this.waterShaders = [];

  for (var i = 0; i < 2; i++) {
    this.waterShaders[i] = new GL.Shader(vertexShader, helperFunctions + '...fragment body...');
  }

}
```

## What changed

In Part 2 we had one simple water shader.

In Part 3 we split that into:

- one shared vertex shader
- one shared helper block
- two fragment shader variants stored in `this.waterShaders`

This is the key organizational change of Part 3.

---

# Step 4 — Delta: Replace the Part 2 Water Shader

## Part 2 `renderer.js`

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

## Part 3 `renderer.js`

```javascript
function Renderer() {
  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  var helperFunctions = '\
    const float IOR_AIR = 1.0;\
    const float IOR_WATER = 1.333;\
    const vec3 ABOVE_WATER_TINT = vec3(0.25, 1.0, 1.25);\
    const vec3 UNDER_WATER_TINT = vec3(0.4, 0.9, 1.0);\
    \
    uniform sampler2D water;\
    uniform samplerCube sky;\
    \
    vec2 intersectBox(vec3 origin, vec3 ray, vec3 boxMin, vec3 boxMax) {\
      vec3 tMin = (boxMin - origin) / ray;\
      vec3 tMax = (boxMax - origin) / ray;\
      vec3 t1 = min(tMin, tMax);\
      vec3 t2 = max(tMin, tMax);\
      float tNear = max(max(t1.x, t1.y), t1.z);\
      float tFar = min(min(t2.x, t2.y), t2.z);\
      return vec2(tNear, tFar);\
    }\
    \
    vec3 poolColor(vec3 point) {\
      vec3 wall = vec3(0.75, 0.85, 0.95);\
      vec3 floor = vec3(0.55, 0.7, 0.9);\
      if (abs(point.y + 1.0) < 0.01) return floor;\
      return wall;\
    }\
    \
    vec3 traceSceneColor(vec3 origin, vec3 ray, vec3 waterTint) {\
      vec2 t = intersectBox(origin, ray, vec3(-1.0, -1.0, -1.0), vec3(1.0, 2.0, 1.0));\
      \
      if (ray.y > 0.0) {\
        vec3 skyColor = textureCube(sky, ray).rgb;\
        return skyColor;\
      }\
      \
      vec3 hit = origin + ray * t.y;\
      return poolColor(hit) * waterTint;\
    }\
  ';

  var vertexShader = '\
    uniform sampler2D water;\
    varying vec3 position;\
    void main() {\
      vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);\
      position = gl_Vertex.xzy;\
      position.y += info.r;\
      gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);\
    }\
  ';

  this.waterShaders = [];

  for (var i = 0; i < 2; i++) {
    this.waterShaders[i] = new GL.Shader(vertexShader, helperFunctions + '\
      uniform vec3 eye;\
      varying vec3 position;\
      \
      vec3 readNormal(vec3 pos) {\
        vec2 coord = pos.xz * 0.5 + 0.5;\
        vec4 info = texture2D(water, coord);\
        \
        for (int j = 0; j < 5; j++) {\
          coord += info.ba * 0.005;\
          info = texture2D(water, coord);\
        }\
        \
        return vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);\
      }\
      \
      void main() {\
        vec3 normal = readNormal(position);\
        vec3 incomingRay = normalize(position - eye);\
      ' + (i ? '\
        normal = -normal;\
        vec3 reflectedRay = reflect(incomingRay, normal);\
        vec3 refractedRay = refract(incomingRay, normal, IOR_WATER / IOR_AIR);\
        float fresnel = mix(0.5, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
        vec3 reflectedColor = traceSceneColor(position, reflectedRay, UNDER_WATER_TINT);\
        vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));\
        gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);\
      ' : '\
        vec3 reflectedRay = reflect(incomingRay, normal);\
        vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);\
        float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
        vec3 reflectedColor = traceSceneColor(position, reflectedRay, ABOVE_WATER_TINT);\
        vec3 refractedColor = traceSceneColor(position, refractedRay, ABOVE_WATER_TINT);\
        gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);\
      ') + '\
      }\
    ');
  }
}
```

---

# Step 5 — Understanding the New `renderer.js`

Let’s break down the new constructor into the parts that matter.

## 5.1 — Shared Helper Block

```javascript
var helperFunctions = '...';
```

This shared GLSL block contains the utility code that both water fragment shaders need.

That includes three helper functions:

- `intersectBox(...)`
- `poolColor(...)`
- `traceSceneColor(...)`

Together they answer a very practical question:

> once we compute a reflected or refracted ray, what color should that ray see?

Instead of putting all of that logic directly inside the main fragment shader, we package it into helpers so the shader can stay focused on the optical steps:

1. reconstruct the normal  
2. compute the view ray  
3. compute reflected and refracted rays  
4. ask the helpers what colors those rays see  
5. blend the results with Fresnel

Let’s look at the three helpers one by one.

---

### `intersectBox(origin, ray, boxMin, boxMax)`

```glsl
vec2 intersectBox(vec3 origin, vec3 ray, vec3 boxMin, vec3 boxMax) {
  vec3 tMin = (boxMin - origin) / ray;
  vec3 tMax = (boxMax - origin) / ray;
  vec3 t1 = min(tMin, tMax);
  vec3 t2 = max(tMin, tMax);
  float tNear = max(max(t1.x, t1.y), t1.z);
  float tFar = min(min(t2.x, t2.y), t2.z);
  return vec2(tNear, tFar);
}
```

This function intersects a ray with an axis-aligned box.

The box is described by two corners:

- `boxMin` = minimum x, y, z
- `boxMax` = maximum x, y, z

The ray is described in parametric form as:

```text
P(t) = origin + ray * t
```

So the goal is to find the values of `t` where the ray enters and exits the box.

#### Step 1 — Intersect the ray with the box planes

```glsl
vec3 tMin = (boxMin - origin) / ray;
vec3 tMax = (boxMax - origin) / ray;
```

For each axis, this computes where the ray hits the two planes on that axis.

For example, along x:

```text
t = (box plane x - ray origin x) / ray direction x
```

The same is done for y and z.

So after these two lines we have:

- one candidate `t` for the “low” plane on each axis
- one candidate `t` for the “high” plane on each axis

#### Step 2 — Sort near and far for each axis

```glsl
vec3 t1 = min(tMin, tMax);
vec3 t2 = max(tMin, tMax);
```

A ray can point in either positive or negative directions, so the smaller value is not always in `tMin`.

These lines sort the two intersection distances per axis into:

- `t1` = near hit on that axis
- `t2` = far hit on that axis

#### Step 3 — Compute the overall entry and exit points

```glsl
float tNear = max(max(t1.x, t1.y), t1.z);
float tFar = min(min(t2.x, t2.y), t2.z);
```

To actually be inside the box, the ray must be inside the x slab, the y slab, and the z slab at the same time.

So:

- the ray **enters** the box at the largest of the three near distances
- the ray **exits** the box at the smallest of the three far distances

That gives:

- `tNear` = first point where the ray is inside all three slabs
- `tFar` = last point where the ray is still inside all three slabs

#### Return value

```glsl
return vec2(tNear, tFar);
```

So the function returns both distances:

- `.x` = entry distance
- `.y` = exit distance

In this tutorial we mainly use `tFar`, because the refracted ray starts near the water surface and continues through the pool until it hits the far interior wall or floor.

---

### `poolColor(point)`

```glsl
vec3 poolColor(vec3 point) {
  vec3 wall = vec3(0.75, 0.85, 0.95);
  vec3 floor = vec3(0.55, 0.7, 0.9);
  if (abs(point.y + 1.0) < 0.01) return floor;
  return wall;
}
```

This function gives a simple color for the pool interior at the point where a ray hits it.

At this stage of the tutorial, we are not yet rendering detailed pool geometry, tiles, or textures.

So instead we fake the pool interior with two flat colors:

- one color for the floor
- one color for the walls

#### Floor test

```glsl
if (abs(point.y + 1.0) < 0.01) return floor;
```

The pool floor is assumed to lie near:

```text
y = -1
```

So this line checks whether the hit point is very close to that height.

Why `+ 1.0`?

Because if `point.y == -1.0`, then:

```text
point.y + 1.0 == 0.0
```

and the absolute value is tiny.

The `0.01` threshold is just a small tolerance so we do not depend on an exact floating-point match.

#### Otherwise, treat it as a wall

```glsl
return wall;
```

If the hit point is not on the floor, we treat it as one of the side walls.

So `poolColor()` is a very simple stand-in for the more detailed pool rendering that comes later.

---

### `traceSceneColor(origin, ray, waterTint)`

```glsl
vec3 traceSceneColor(vec3 origin, vec3 ray, vec3 waterTint) {
  vec2 t = intersectBox(origin, ray, vec3(-1.0, -1.0, -1.0), vec3(1.0, 2.0, 1.0));

  if (ray.y > 0.0) {
    vec3 skyColor = textureCube(sky, ray).rgb;
    return skyColor;
  }

  vec3 hit = origin + ray * t.y;
  return poolColor(hit) * waterTint;
}
```

This is the main scene-query helper.

Its job is:

- take a ray
- decide whether it sees the sky or the pool interior
- return the corresponding color

#### Step 1 — Intersect the ray with the pool box

```glsl
vec2 t = intersectBox(origin, ray, vec3(-1.0, -1.0, -1.0), vec3(1.0, 2.0, 1.0));
```

This defines a simple box-shaped pool volume.

So now we know where the ray would enter and leave that volume.

#### Step 2 — Upward rays see the sky

```glsl
if (ray.y > 0.0) {
  vec3 skyColor = textureCube(sky, ray).rgb;
  return skyColor;
}
```

If the ray points upward, we treat it as looking out into the environment.

So we sample the cubemap in the ray direction and return that color.

This is how reflected rays produce sky reflections.

#### Step 3 — Downward rays hit the pool interior

```glsl
vec3 hit = origin + ray * t.y;
```

If the ray points downward, we assume it travels into the pool.

We already know the entry and exit distances from `intersectBox()`.

Here we use `t.y`, the far intersection distance, to find the point where the ray exits the pool box on the opposite interior surface.

That gives us the hit point in 3D.

#### Step 4 — Color the interior surface and tint it

```glsl
return poolColor(hit) * waterTint;
```

Finally, we evaluate the pool color at that hit point and multiply by a tint.

That tint lets the same helper be used in slightly different optical situations:

- above-water rays can tint the pool view one way
- underwater rays can tint it another way

So `traceSceneColor()` is the bridge between abstract rays and visible scene color.

It is the function that lets the water shader say:

- “What does the reflected ray see?”
- “What does the refracted ray see?”

without having to repeat all the scene-intersection logic inline.

---

### Why these helpers matter

These three helpers work together as a small ray-tracing toolkit inside the fragment shader:

- `intersectBox()` finds where a ray crosses the pool volume
- `poolColor()` assigns a simple material color to the hit surface
- `traceSceneColor()` decides whether the ray sees sky or pool and returns the final color

That is what makes the reflection and refraction code manageable.

The main shader computes the rays.

The helper block answers what those rays hit.

---

## 5.2 — Shared Vertex Shader

In Part 2, the water renderer used a single shader program that did both:

- vertex displacement
- simple lighting setup

The vertex shader looked like this:

```javascript
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
', '...fragment shader...');
```

That made sense in Part 2, because the fragment shader was very simple. It just needed a normal and a light direction for Lambert shading.

In Part 3, the fragment shader becomes much more sophisticated. It now needs to compute:

- reflection
- refraction
- Fresnel blending
- different behavior above and below the surface

That changes what the fragment shader needs as input.

Instead of receiving a normal computed in the vertex shader, the Part 3 fragment shader re-samples the water texture and reconstructs the normal per-fragment. That is more accurate for optical effects, because reflection and refraction are very sensitive to the exact normal direction.

So the vertex shader gets simpler.

---

### Part 2 vertex shader

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

### Part 3 vertex shader

```glsl
uniform sampler2D water;
varying vec3 position;

void main() {
  vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);
  position = gl_Vertex.xzy;
  position.y += info.r;
  gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);
}
```

---

### Code delta

Here is the exact conceptual delta from Part 2 to Part 3:

#### Part 2

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

#### Part 3

```glsl
uniform sampler2D water;
varying vec3 position;

void main() {
  vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);
  position = gl_Vertex.xzy;
  position.y += info.r;
  gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);
}
```

#### What changed

We removed:

```glsl
varying vec3 normal;
normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
```

Everything else stays the same.

---

### Why remove the normal from the vertex shader?

In Part 2, reconstructing the normal in the vertex shader was good enough because the fragment shader only used it for smooth diffuse lighting.

But in Part 3, the normal affects:

- the reflected ray direction
- the refracted ray direction
- the Fresnel term

Those are all nonlinear calculations. If we compute the normal per-vertex and then interpolate it across the triangle, we lose detail and can get visibly softer or less accurate optical behavior.

So in Part 3, the vertex shader focuses only on geometry:

1. sample the water height
2. displace the mesh
3. pass the displaced position to the fragment shader

Then the fragment shader uses that position to do a fresh water-texture lookup and reconstruct the normal there.

That gives better-looking reflection and refraction.

---

## 5.3 — Two Fragment Shaders Built in a Loop

This is the biggest structural change in Part 3.

In Part 2, the renderer created one water shader:

```javascript
this.waterShader = new GL.Shader(vertexShaderSource, fragmentShaderSource);
```

That was enough because the fragment shader only did one job: simple directional lighting.

But in Part 3, water needs to behave differently depending on which side of the surface we are shading.

From above the surface:

- the ray starts in air
- the refraction ratio is air → water
- the water tends to look more transparent straight on and more reflective at grazing angles

From below the surface:

- the ray starts in water
- the refraction ratio is water → air
- the normal orientation needs to be flipped
- the underwater appearance is tinted differently

We could write one giant fragment shader full of conditionals, but that would make the code harder to read and harder to explain.

Instead, we build two closely related shaders in a loop.

---

### Part 2 shader structure

In Part 2 the renderer looked like this:

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

That is one mesh and one shader program.

---

### Part 3 shader structure

In Part 3, the renderer changes to this:

```javascript
this.waterShaders = [];

for (var i = 0; i < 2; i++) {
  this.waterShaders[i] = new GL.Shader(vertexShader, helperFunctions + '...fragment body...');
}
```

That means:

- `this.waterShaders[0]` = one side of the water
- `this.waterShaders[1]` = the other side of the water

The shaders share:

- the same vertex shader
- the same helper functions
- most of the same fragment logic

They differ only in the small section of code that depends on which side of the interface we are rendering.

---

### Code delta: one shader becomes two

#### Part 2

```javascript
this.waterShader = new GL.Shader(vertexShaderSource, fragmentShaderSource);
```

#### Part 3

```javascript
this.waterShaders = [];

for (var i = 0; i < 2; i++) {
  this.waterShaders[i] = new GL.Shader(vertexShader, helperFunctions + '...fragment body...');
}
```

#### What changed

Instead of one shader object:

```javascript
this.waterShader
```

we now create an array:

```javascript
this.waterShaders
```

And instead of writing one fragment shader, we generate two variants.

---

### Full Part 3 pattern

Here is the full structure again:

```javascript
this.waterShaders = [];

for (var i = 0; i < 2; i++) {
  this.waterShaders[i] = new GL.Shader(vertexShader, helperFunctions + '\
    uniform vec3 eye;\
    varying vec3 position;\
    \
    vec3 readNormal(vec3 pos) {\
      vec2 coord = pos.xz * 0.5 + 0.5;\
      vec4 info = texture2D(water, coord);\
      \
      for (int j = 0; j < 5; j++) {\
        coord += info.ba * 0.005;\
        info = texture2D(water, coord);\
      }\
      \
      return vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);\
    }\
    \
    void main() {\
      vec3 normal = readNormal(position);\
      vec3 incomingRay = normalize(position - eye);\
    ' + (i ? '\
      normal = -normal;\
      vec3 reflectedRay = reflect(incomingRay, normal);\
      vec3 refractedRay = refract(incomingRay, normal, IOR_WATER / IOR_AIR);\
      float fresnel = mix(0.5, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
      vec3 reflectedColor = traceSceneColor(position, reflectedRay, UNDER_WATER_TINT);\
      vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));\
      gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);\
    ' : '\
      vec3 reflectedRay = reflect(incomingRay, normal);\
      vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);\
      float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
      vec3 reflectedColor = traceSceneColor(position, reflectedRay, ABOVE_WATER_TINT);\
      vec3 refractedColor = traceSceneColor(position, refractedRay, ABOVE_WATER_TINT);\
      gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);\
    ') + '\
    }\
  ');
}
```

At first glance this can look intimidating, so it helps to break it into layers.

---

### Layer 1 — Shared fragment code

The first part of the fragment shader is the same in both variants:

```glsl
uniform vec3 eye;
varying vec3 position;

vec3 readNormal(vec3 pos) {
  vec2 coord = pos.xz * 0.5 + 0.5;
  vec4 info = texture2D(water, coord);

  for (int j = 0; j < 5; j++) {
    coord += info.ba * 0.005;
    info = texture2D(water, coord);
  }

  return vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
}

void main() {
  vec3 normal = readNormal(position);
  vec3 incomingRay = normalize(position - eye);
  ...
}
```

This shared code does three important things.

#### 1. It receives the eye position

```glsl
uniform vec3 eye;
```

The fragment shader needs the camera position in order to compute the incoming viewing ray.

#### 2. It reconstructs the normal per-fragment

```glsl
vec3 normal = readNormal(position);
```

This replaces the old Part 2 approach of passing a normal from the vertex shader.

The helper `readNormal()` samples the water texture using the displaced surface position and reconstructs the normal there.

That gives more accurate normals for reflection and refraction.

#### 3. It computes the incoming viewing ray

```glsl
vec3 incomingRay = normalize(position - eye);
```

This ray points from the eye toward the surface point being shaded.

That is the ray we feed into `reflect()` and `refract()`.

---

### Why `readNormal()` exists

This helper was not needed in Part 2.

In Part 3 it appears because optical shading needs more precise normals than simple diffuse lighting.

The function does:

```glsl
vec2 coord = pos.xz * 0.5 + 0.5;
vec4 info = texture2D(water, coord);
```

which maps the displaced water-surface position back into water-texture coordinates.

Then it does a few small offset steps:

```glsl
for (int j = 0; j < 5; j++) {
  coord += info.ba * 0.005;
  info = texture2D(water, coord);
}
```

The loop does not change the simulated water height. It only changes **where we sample the normal from for shading**. At each step, the lookup coordinate is nudged a little in the horizontal direction indicated by the current normal, then the texture is sampled again. That means the shader is not using the normal exactly at the current point, but a normal gathered slightly farther along the local slope. Visually, this biases the shading toward stronger slope changes near wave crests, so reflections and highlights transition more tightly instead of being spread smoothly across a broad rounded hump. In that sense the waves look **sharper**.

After that it reconstructs the full normal:

```glsl
return vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
```

So `readNormal()` is a fragment-side replacement for the simpler normal reconstruction that Part 2 did in the vertex shader.

---

### Layer 2 — Above-water branch

When `i == 0`, the loop inserts this body:

```glsl
vec3 reflectedRay = reflect(incomingRay, normal);
vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);
float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));
vec3 reflectedColor = traceSceneColor(position, reflectedRay, ABOVE_WATER_TINT);
vec3 refractedColor = traceSceneColor(position, refractedRay, ABOVE_WATER_TINT);
gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);
```

This is the top-of-water case.

#### Why it changed from Part 2

In Part 2, the fragment shader simply did:

```glsl
float diffuse = max(0.0, dot(normalize(normal), normalize(light)));
vec3 color = vec3(0.2, 0.6, 1.0) * (0.2 + 0.8 * diffuse);
gl_FragColor = vec4(color, 1.0);
```

That was a lighting model.

In Part 3, we replace that with an optical model:

- compute reflected direction
- compute refracted direction
- look up colors for both
- blend them using Fresnel

#### Why `IOR_AIR / IOR_WATER`?

Because in this pass the ray starts in air and bends into water.

That is the physical direction of transmission for the above-water view.

#### Why mix refraction and reflection?

Looking straight down at water, you usually see more into the water.

Looking across it at a shallow angle, you see stronger reflections.

That angle-dependent blend is one of the biggest visual cues that a surface is water.

---

### Layer 3 — Underwater branch

When `i == 1`, the loop inserts this body:

```glsl
normal = -normal;
vec3 reflectedRay = reflect(incomingRay, normal);
vec3 refractedRay = refract(incomingRay, normal, IOR_WATER / IOR_AIR);
float fresnel = mix(0.5, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));
vec3 reflectedColor = traceSceneColor(position, reflectedRay, UNDER_WATER_TINT);
vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));
gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);
```

This is the underside-of-water case.

#### Why flip the normal?

The water surface normal was reconstructed in its default orientation, appropriate for the top surface.

When shading the underside, we want the normal to face the other way, so we flip it:

```glsl
normal = -normal;
```

#### Why reverse the IOR ratio?

Now the ray is traveling from water toward air, so the transmission ratio changes to:

```glsl
IOR_WATER / IOR_AIR
```

That is the physically appropriate direction for the underwater pass.

#### Why does this branch look a little different?

Underwater visuals are not just the same thing with the camera moved below the surface.

The balance between reflected and refracted light changes, and the tint is different.

So this branch uses:

- a different Fresnel baseline
- a different tint
- a slightly different blend expression

That is exactly the kind of difference that would make a single giant shader harder to read.

---

### Why build the shaders in a loop?

Because most of the code is identical.

Both shader variants need:

- the same `eye` uniform
- the same `position` varying
- the same `readNormal()` helper
- the same reflection/refraction structure
- the same helper block

Only a small middle section changes:

- normal flip or not
- air→water vs water→air
- above-water tint vs underwater tint
- slightly different Fresnel and final blend

So instead of duplicating dozens of lines twice, we generate the two versions programmatically.

That makes the code shorter while still keeping the two cases visibly separate.

---

### Why this is better than branching inside one shader

You might wonder why not do something like:

```glsl
if (underwater) {
  ...
} else {
  ...
}
```

That would work, but this tutorial intentionally follows the structure of the reference code, where the renderer builds multiple related shaders by combining shared GLSL with small variant-specific sections.

That has a few advantages:

- the above-water and underwater cases stay clearly separated
- shared code is not duplicated
- the render loop can select the shader by pass
- it matches the mental model of “two-sided water rendering”

So the renderer organization becomes:

- shared vertex shader
- shared helper block
- two fragment shader variants
- two render passes with opposite face culling

That is the core architectural change in Part 3.

---

### Summary of what changed from Part 2

Part 2 used one water shader with a simple fragment lighting model.

Part 3 replaces that with two shaders built from shared pieces.

#### Part 2

- one shader
- normal reconstructed in vertex shader
- fragment shader does Lambert lighting

#### Part 3

- one shared vertex shader
- one shared helper block
- two fragment shader variants
- normal reconstructed in fragment shader
- fragment shader computes reflection, refraction, and Fresnel
- render loop draws the water twice, once per side

That is why this section matters so much.

It is the point where the renderer stops being “draw a blue displaced mesh” and becomes “shade a refractive reflective interface.”

---

# Step 6 — Delta: Part 2 `renderWater()` vs Part 3 `renderWater()`

## Part 2

```javascript
Renderer.prototype.renderWater = function(water) {
  water.textureA.bind(0);
  this.waterShader.uniforms({
    water: 0,
    light: [2, 2, -1]
  }).draw(this.waterMesh);
};
```

## Part 3

```javascript
Renderer.prototype.renderWater = function(water, sky) {
  var tracer = new GL.Raytracer();

  water.textureA.bind(0);
  sky.bind(1);

  gl.enable(gl.CULL_FACE);

  for (var i = 0; i < 2; i++) {
    gl.cullFace(i ? gl.BACK : gl.FRONT);

    this.waterShaders[i].uniforms({
      water: 0,
      sky: 1,
      eye: tracer.eye
    }).draw(this.waterMesh);
  }

  gl.disable(gl.CULL_FACE);
};
```

## What changed

In Part 2 we:

- bound one texture
- used one shader
- drew once

In Part 3 we:

- bind the water texture
- bind the sky cubemap
- compute the eye position with `GL.Raytracer()`
- enable culling
- draw the same mesh twice with different cull modes and different shaders

This is the exact place where the two-sided rendering happens.

---

# Step 7 — The View Ray and Why `eye` Is Needed

The fragment shader needs the camera position to compute the incoming viewing ray.

That is why `renderWater()` now creates:

```javascript
var tracer = new GL.Raytracer();
```

and passes:

```javascript
eye: tracer.eye
```

Then in the fragment shader we compute:

```glsl
vec3 incomingRay = normalize(position - eye);
```

This gives the ray from the eye toward the water surface point being shaded.

Once we have that ray, we can compute:

- `reflect(incomingRay, normal)`
- `refract(incomingRay, normal, eta)`

---

# Step 8 — Why the Normal Is Re-read in the Fragment Shader

In Part 2 we reconstructed the normal in the vertex shader.

In Part 3 we move that work into the fragment shader.

Why?

Because the optical effect is very sensitive to the exact normal direction.

Reflection and refraction need **per-fragment** normals, not just interpolated per-vertex normals.

So the fragment shader samples the water texture again, reconstructs the normal there, and uses it for the optical calculations.

---

# Step 9 — Above-Water vs Underwater Bodies

Inside the loop, the shader body changes depending on `i`.

## Above-water branch

```glsl
vec3 reflectedRay = reflect(incomingRay, normal);
vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);
float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));
vec3 reflectedColor = traceSceneColor(position, reflectedRay, ABOVE_WATER_TINT);
vec3 refractedColor = traceSceneColor(position, refractedRay, ABOVE_WATER_TINT);
gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);
```

This is the top-of-water view.

The ray starts in air and enters water, so the refraction ratio is:

```glsl
IOR_AIR / IOR_WATER
```

---

## Underwater branch

```glsl
normal = -normal;
vec3 reflectedRay = reflect(incomingRay, normal);
vec3 refractedRay = refract(incomingRay, normal, IOR_WATER / IOR_AIR);
float fresnel = mix(0.5, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));
vec3 reflectedColor = traceSceneColor(position, reflectedRay, UNDER_WATER_TINT);
vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));
gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);
```

Now the ray starts in water and exits into air.

So:

- the normal is flipped
- the IOR ratio is reversed
- the tint and blend are slightly different

That is why the two-pass design exists.

---

# Full Part 3 Files

Below are the complete files after this part.

---

## `index.html`

```html
<!DOCTYPE html>
<html>
<head>
  <script src="lightgl.js"></script>
  <script src="cubemap.js"></script>
  <script src="water.js"></script>
  <script src="renderer.js"></script>
  <script src="main.js"></script>
  <style>
    html, body { margin: 0; height: 100%; overflow: hidden; background: black; }
    canvas { display: block; width: 100%; height: 100%; }
    img { display: none; }
  </style>
</head>
<body>
  <img id="xneg" src="xneg.jpg">
  <img id="xpos" src="xpos.jpg">
  <img id="ypos" src="ypos.jpg">
  <img id="zneg" src="zneg.jpg">
  <img id="zpos" src="zpos.jpg">
</body>
</html>
```

---

## `water.js`

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

## `renderer.js`

```javascript
function Renderer() {
  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  var helperFunctions = '\
    const float IOR_AIR = 1.0;\
    const float IOR_WATER = 1.333;\
    const vec3 ABOVE_WATER_TINT = vec3(0.25, 1.0, 1.25);\
    const vec3 UNDER_WATER_TINT = vec3(0.4, 0.9, 1.0);\
    \
    uniform sampler2D water;\
    uniform samplerCube sky;\
    \
    vec2 intersectBox(vec3 origin, vec3 ray, vec3 boxMin, vec3 boxMax) {\
      vec3 tMin = (boxMin - origin) / ray;\
      vec3 tMax = (boxMax - origin) / ray;\
      vec3 t1 = min(tMin, tMax);\
      vec3 t2 = max(tMin, tMax);\
      float tNear = max(max(t1.x, t1.y), t1.z);\
      float tFar = min(min(t2.x, t2.y), t2.z);\
      return vec2(tNear, tFar);\
    }\
    \
    vec3 poolColor(vec3 point) {\
      vec3 wall = vec3(0.75, 0.85, 0.95);\
      vec3 floor = vec3(0.55, 0.7, 0.9);\
      if (abs(point.y + 1.0) < 0.01) return floor;\
      return wall;\
    }\
    \
    vec3 traceSceneColor(vec3 origin, vec3 ray, vec3 waterTint) {\
      vec2 t = intersectBox(origin, ray, vec3(-1.0, -1.0, -1.0), vec3(1.0, 2.0, 1.0));\
      \
      if (ray.y > 0.0) {\
        vec3 skyColor = textureCube(sky, ray).rgb;\
        return skyColor;\
      }\
      \
      vec3 hit = origin + ray * t.y;\
      return poolColor(hit) * waterTint;\
    }\
  ';

  var vertexShader = '\
    uniform sampler2D water;\
    varying vec3 position;\
    void main() {\
      vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);\
      position = gl_Vertex.xzy;\
      position.y += info.r;\
      gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);\
    }\
  ';

  this.waterShaders = [];

  for (var i = 0; i < 2; i++) {
    this.waterShaders[i] = new GL.Shader(vertexShader, helperFunctions + '\
      uniform vec3 eye;\
      varying vec3 position;\
      \
      vec3 readNormal(vec3 pos) {\
        vec2 coord = pos.xz * 0.5 + 0.5;\
        vec4 info = texture2D(water, coord);\
        \
        for (int j = 0; j < 5; j++) {\
          coord += info.ba * 0.005;\
          info = texture2D(water, coord);\
        }\
        \
        return vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);\
      }\
      \
      void main() {\
        vec3 normal = readNormal(position);\
        vec3 incomingRay = normalize(position - eye);\
      ' + (i ? '\
        normal = -normal;\
        vec3 reflectedRay = reflect(incomingRay, normal);\
        vec3 refractedRay = refract(incomingRay, normal, IOR_WATER / IOR_AIR);\
        float fresnel = mix(0.5, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
        vec3 reflectedColor = traceSceneColor(position, reflectedRay, UNDER_WATER_TINT);\
        vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));\
        gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);\
      ' : '\
        vec3 reflectedRay = reflect(incomingRay, normal);\
        vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);\
        float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
        vec3 reflectedColor = traceSceneColor(position, reflectedRay, ABOVE_WATER_TINT);\
        vec3 refractedColor = traceSceneColor(position, refractedRay, ABOVE_WATER_TINT);\
        gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);\
      ') + '\
      }\
    ');
  }
}

Renderer.prototype.renderWater = function(water, sky) {
  var tracer = new GL.Raytracer();

  water.textureA.bind(0);
  sky.bind(1);

  gl.enable(gl.CULL_FACE);

  for (var i = 0; i < 2; i++) {
    gl.cullFace(i ? gl.BACK : gl.FRONT);

    this.waterShaders[i].uniforms({
      water: 0,
      sky: 1,
      eye: tracer.eye
    }).draw(this.waterMesh);
  }

  gl.disable(gl.CULL_FACE);
};
```

---

## `main.js`

```javascript
var gl = GL.create();

var water;
var renderer;
var cubemap;

var angleX = -25;
var angleY = -200.5;

window.onload = function() {

  water = new Water();
  renderer = new Renderer();

  cubemap = new Cubemap({
    xneg: document.getElementById('xneg'),
    xpos: document.getElementById('xpos'),
    yneg: document.getElementById('ypos'),
    ypos: document.getElementById('ypos'),
    zneg: document.getElementById('zneg'),
    zpos: document.getElementById('zpos')
  });

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

  gl.disable(gl.DEPTH_TEST);

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

  renderer.renderWater(water, cubemap);

}
```

---

# How the Pieces Fit Together

## In `main.js`

You create:

- `water`
- `renderer`
- `cubemap`

Then each frame you:

- advance the simulation
- update normals
- set the camera transform
- call `renderer.renderWater(water, cubemap)`

## In `renderWater()`

You bind:

- the water simulation texture
- the sky cubemap

Then you render the same mesh twice:

- first with one cull mode and one shader
- then with the opposite cull mode and the other shader

## In the fragment shader

You:

- reconstruct the normal
- compute the incoming view ray
- compute reflected and refracted rays
- trace them
- Fresnel blend the two colors

That is the complete Part 3 pipeline.

---

# Result of Part 3

At the end of this stage you should see:

- sky reflections on the water
- refracted view into the pool interior
- stronger reflection at glancing angles
- different appearance above and below the surface

It still is not the full final demo yet.

What is still missing is the rest of the scene system:

- explicit pool rendering
- the sphere
- caustics
- more complete shadowing and scene interaction

Those are the pieces that arrive in the following parts.

