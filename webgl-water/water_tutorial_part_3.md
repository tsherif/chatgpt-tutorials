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

## 5.1 — Shared helper block

```javascript
var helperFunctions = '...';
```

This block is prepended to both fragment shaders.

That is where we define:

- IOR constants
- box intersection logic
- a helper that decides what color a ray sees

This avoids duplicating those definitions in both shader variants.

---

## 5.2 — Shared vertex shader

```javascript
var vertexShader = '...';
```

This still does only water displacement.

That part did **not** fundamentally change from Part 2.

It still:

- samples the water texture
- remaps the plane from XY into XZ
- adds `info.r` to `position.y`
- projects the displaced vertex

---

## 5.3 — Two fragment shaders built in a loop

```javascript
this.waterShaders = [];

for (var i = 0; i < 2; i++) {
  this.waterShaders[i] = new GL.Shader(...);
}
```

Instead of writing two nearly identical shaders by hand, we construct them in a loop.

The shared parts are written once.

Then the code inserts one of two small side-specific bodies:

- `i == 0` → above-water logic
- `i == 1` → underwater logic

That is why `this.waterShaders` is an array.

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

