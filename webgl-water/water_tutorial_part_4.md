# WebGL Water Tutorial — Part 4: Pool Geometry and the Sphere

In Part 3, the water surface began behaving like a real optical interface.

However, the reflection and refraction rays were still interacting mostly with a **placeholder scene**:

- a simple sky cubemap
- a box-intersection helper
- flat placeholder colors for the pool interior

So even though the optics were working, the scene was still mostly imaginary.

In this part we introduce the real scene geometry:

- the **pool walls and floor**
- the **sphere inside the pool**

These objects are rendered directly, and the water shader’s reflection and refraction rays are updated so they can interact with those same objects.

At the end of Part 4, the scene becomes much more spatially coherent.

The water is no longer just reflecting and refracting an abstract box.  
It is now tied to the actual pool and the actual sphere that the renderer draws.

---

# What Changes in Part 4

The simulation still stays exactly the same.

`water.js` is unchanged.

What changes is the renderer:

- we add state for the sphere
- we add real meshes for the sphere and pool
- we add a tile texture for the pool walls and floor
- we expand the shared GLSL helper block so rays can intersect the sphere
- we replace the placeholder pool-color helper with real wall/floor shading
- we add dedicated shaders and render methods for the sphere and the pool
- we update the draw order in `main.js`

So Part 4 is the stage where the renderer stops being “water over an implied scene” and becomes “water in a real scene.”

---

# Architecture of the Scene

All scene elements still depend on the **same water simulation texture**.

```text
water simulation
(height + velocity + normals)
          │
          ▼
   shared water texture
          │
  ┌───────┼─────────┐
  ▼       ▼         ▼
water   pool     sphere
shader  shader    shader
```

Because all three materials read from the same water texture:

- they agree on the water height
- they agree on the surface normals
- they respond consistently as the waves move

That shared state is the key architectural idea.

The water surface, the submerged sphere, and the pool all stay visually synchronized because they are all reading the same simulation result.

---

# Files Modified in This Part

```text
renderer.js
main.js
```

Unchanged:

```text
water.js
```

---

# Code Delta Overview

## `renderer.js`

### Part 3
- water mesh
- cubemap helpers
- two-sided water shader
- `renderWater()`

### Part 4
- add sphere state
- add `sphereMesh`
- add `cubeMesh` for the pool
- add tile texture
- extend shared GLSL helpers
- add sphere shader
- add pool shader
- add `renderSphere()`
- add `renderCube()`
- update `renderWater()` uniforms so the water shader knows about the sphere

## `main.js`

### Part 3
- create `water`
- create `renderer`
- create `cubemap`
- advance the simulation
- render water only

### Part 4
- create sphere position/radius state
- pass sphere state into the renderer each frame
- render pool first
- render water second
- render sphere last

---

# Step 1 — Delta: Add Sphere State to the Renderer

In Part 3, the renderer had no notion of scene objects other than the water surface and cubemap.

In Part 4, we add explicit state for the sphere.

## Add inside the `Renderer` constructor

```javascript
this.sphereCenter = new GL.Vector(0, -0.5, 0);
this.sphereRadius = 0.25;
```

## What changed

This is new state owned by the renderer.

The water shader will use it when tracing reflection and refraction rays.

The sphere shader will also use it when positioning the sphere mesh.

So these values become shared scene parameters:

- where the sphere is
- how large it is

Later parts will animate or otherwise control these values from the application.

---

# Step 2 — Delta: Add the New Scene Meshes

Part 3 only needed one mesh for the water surface:

```javascript
this.waterMesh = GL.Mesh.plane({ detail: 200 });
```

Part 4 adds two more.

## Add inside the `Renderer` constructor

```javascript
this.sphereMesh = GL.Mesh.sphere({ detail: 10 });

this.cubeMesh = GL.Mesh.cube();
this.cubeMesh.triangles.splice(4, 2);
this.cubeMesh.compile();
```

---

## What the new meshes are for

### `sphereMesh`

This is the geometry for the ball inside the pool.

It starts as a unit sphere centered at the origin.

The sphere shader will scale and translate it using:

- `sphereRadius`
- `sphereCenter`

### `cubeMesh`

This becomes the pool interior.

A cube is a good starting primitive because the pool is box-shaped.

But a normal cube has six faces, and a pool must be open at the top.

So we remove the top face.

---

## Why remove two triangles?

Each cube face is made of two triangles.

So:

```javascript
this.cubeMesh.triangles.splice(4, 2);
```

removes one face.

That gives us an open-top cube, which we can then reshape vertically into pool dimensions.

The call to `cubeMesh.compile()` uploads the new data to the GPU.

So the sequence is:

```text
cube
  ↓
remove top face
  ↓
open box
  ↓
use as pool interior
```

---

# Step 3 — Delta: Add the Pool Tile Texture

In Part 3, the pool interior was only represented by placeholder colors inside the helper block.

In Part 4, the pool is real scene geometry, so it gets a real texture.

## Add inside the `Renderer` constructor

```javascript
this.tileTexture = GL.Texture.fromImage(
  document.getElementById('tiles'),
  {
    minFilter: gl.LINEAR_MIPMAP_LINEAR,
    wrap: gl.REPEAT
  }
);
```

---

## What changed

Before this, the pool interior looked like this conceptually:

- wall = flat color
- floor = flat color

Now the pool shader can sample a real texture.

That makes the pool walls and floor feel like physical surfaces instead of a placeholder.

### Why use mipmaps and repeat wrap?

- `gl.LINEAR_MIPMAP_LINEAR` gives better minification when the pool recedes in the distance
- `gl.REPEAT` lets the tiles repeat across the surfaces instead of stretching once over the entire wall

So this is the first time the scene gets a true material texture.

---

# Step 4 — Delta: Extend the Shared Helper Block

In Part 3, the shared helper block handled:

- box intersection
- placeholder pool color
- tracing a ray to either the sky or the box-shaped pool

Now the helper block must also support the sphere.

That means the water shader’s reflected and refracted rays can hit:

- the sky
- the pool walls/floor
- the sphere

So the helper block grows substantially in Part 4.

---

## Part 3 helper responsibilities

Part 3 had helper functions like:

- `intersectBox(...)`
- `poolColor(...)`
- `traceSceneColor(...)`

These were enough for a fake pool interior.

---

## Part 4 helper responsibilities

Now we add:

- `intersectSphere(...)`
- `getSphereColor(...)`
- `getWallColor(...)`

And we update `traceSceneColor(...)` so it can choose between:

- sphere hit
- wall/floor hit
- sky lookup

That is the architectural shift in Part 4:

the helper block stops representing an imaginary scene and starts representing the actual rendered scene.

---

# Step 5 — Add Ray–Sphere Intersection

The water shader needs to know whether a reflected or refracted ray hits the sphere before it hits the pool.

So we add a ray–sphere intersection helper.

## Add this to the shared GLSL helper block

```glsl
float intersectSphere(
  vec3 origin,
  vec3 ray,
  vec3 center,
  float radius
) {
  vec3 oc = origin - center;

  float a = dot(ray, ray);
  float b = 2.0 * dot(oc, ray);
  float c = dot(oc, oc) - radius * radius;

  float d = b * b - 4.0 * a * c;

  if (d < 0.0) return 1e6;

  float t = (-b - sqrt(d)) / (2.0 * a);

  return t > 0.0 ? t : 1e6;
}
```

---

## What this function does

It solves the standard quadratic equation for ray–sphere intersection.

The ray is:

```text
origin + ray * t
```

The sphere is all points whose distance from `center` is `radius`.

Substituting the ray into the sphere equation gives a quadratic in `t`.

### Meaning of the discriminant

```glsl
float d = b * b - 4.0 * a * c;
```

- `d < 0` means the ray misses the sphere
- `d >= 0` means the ray intersects it

If the ray misses, we return a very large number:

```glsl
1e6
```

That acts like “no hit.”

If the ray hits, we return the nearest positive hit distance.

So the function tells the shader:

> how far along this ray do I reach the sphere?

---

# Step 6 — Add the Sphere Shading Helper

In Part 4, the sphere becomes a real object in the scene, not just a conceptual target.

So we give it its own shading helper.

## Add this to the shared GLSL helper block

```glsl
vec3 getSphereColor(vec3 point) {
  vec3 color = vec3(0.5);

  vec3 normal = (point - sphereCenter) / sphereRadius;

  float diffuse = max(0.0, dot(normal, -light)) * 0.5;

  color += diffuse;

  return color;
}
```

---

## What this function does

This is intentionally simple lighting.

The sphere is mostly there to make the scene readable through refraction and reflection.

### Step 1 — Base color

```glsl
vec3 color = vec3(0.5);
```

The sphere starts as a neutral gray.

### Step 2 — Sphere normal

```glsl
vec3 normal = (point - sphereCenter) / sphereRadius;
```

For a sphere, the outward normal is easy to compute:

- subtract the center
- divide by the radius

That gives a unit-length direction from the sphere center to the surface point.

### Step 3 — Diffuse lighting

```glsl
float diffuse = max(0.0, dot(normal, -light)) * 0.5;
color += diffuse;
```

This gives the sphere simple directional shading.

So the sphere is not meant to be fancy here.

It is meant to be:

- easy to see directly
- easy to identify through the water
- something meaningful for water rays to hit

---

# Step 7 — Replace Placeholder Pool Coloring with Real Wall Shading

In Part 3, the helper block had a simple placeholder like:

```glsl
vec3 poolColor(vec3 point) { ... }
```

That was enough when the pool was still only conceptual.

In Part 4, we replace that with a more detailed wall/floor shading function.

## Add this to the shared GLSL helper block

```glsl
vec3 getWallColor(vec3 point) {
  vec3 normal;
  vec3 color;

  if (abs(point.x) > 0.999) {
    normal = vec3(-point.x, 0.0, 0.0);
    color = texture2D(tiles, point.yz * 0.5 + vec2(1.0, 0.5)).rgb;
  } else if (abs(point.z) > 0.999) {
    normal = vec3(0.0, 0.0, -point.z);
    color = texture2D(tiles, point.yx * 0.5 + vec2(1.0, 0.5)).rgb;
  } else {
    normal = vec3(0.0, 1.0, 0.0);
    color = texture2D(tiles, point.xz * 0.5 + 0.5).rgb;
  }

  float diffuse = max(0.0, dot(normal, -light));

  return color * (0.5 + diffuse * 0.5);
}
```

---

## What this function does

It determines which pool surface was hit:

- x wall
- z wall
- floor

and then:

- assigns the correct surface normal
- samples the tile texture with the appropriate coordinate mapping
- applies simple diffuse lighting

---

## Why detect the wall this way?

The pool interior is a box, so any hit point lies near one of its boundaries.

### Side walls in x

```glsl
if (abs(point.x) > 0.999)
```

If `x` is near `±1`, then the point is on one of the side walls.

The wall normal points inward/outward along x.

### Side walls in z

```glsl
else if (abs(point.z) > 0.999)
```

If `z` is near `±1`, then the point is on the other pair of walls.

### Otherwise, it is the floor

If the point is not on an x wall or z wall, we treat it as the pool floor.

---

## Why use different texture coordinates on each face?

Each surface needs a UV mapping that makes sense for its orientation.

So:

- x walls use `point.yz`
- z walls use `point.yx`
- floor uses `point.xz`

This lets the tile pattern align naturally with each face.

That is why this helper is much more detailed than the old Part 3 placeholder.

It is now really shading a box-shaped tiled pool.

---

# Step 8 — Update `traceSceneColor()` to Use the Real Scene

Now that we have real scene helpers, the ray tracer inside the water shader must be updated.

Instead of only deciding between sky and a generic pool box, it must decide what object a ray hits first.

## Replace the old Part 3-style `traceSceneColor(...)` with the Part 4 version

```glsl
vec3 traceSceneColor(vec3 origin, vec3 ray, vec3 waterTint) {
  float tSphere = intersectSphere(origin, ray, sphereCenter, sphereRadius);
  vec2 tCube = intersectBox(origin, ray, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));

  if (tSphere < tCube.y) {
    vec3 hit = origin + ray * tSphere;
    return getSphereColor(hit) * waterTint;
  }

  if (ray.y < 0.0) {
    vec3 hit = origin + ray * tCube.y;
    return getWallColor(hit) * waterTint;
  }

  return textureCube(sky, ray).rgb;
}
```

---

## What changed

Part 3 did this roughly:

- upward ray → sky
- downward ray → pool box

Part 4 now does this:

1. intersect the sphere
2. intersect the pool
3. if the sphere is hit first, shade the sphere
4. otherwise if the ray goes downward, shade the pool
5. otherwise sample the sky

So the water shader is now tracing a ray through a real scene model.

This is what makes the refraction and reflection feel anchored.

---

# Step 9 — Add the Sphere Shader

The sphere is now an explicitly rendered object, so it needs its own shader.

---

## Sphere vertex shader

```glsl
varying vec3 position;

void main() {
  position = sphereCenter + gl_Vertex.xyz * sphereRadius;
  gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);
}
```

---

## What the sphere vertex shader does

The input `gl_Vertex` comes from the unit sphere mesh.

So the shader:

1. scales the unit sphere by `sphereRadius`
2. translates it by `sphereCenter`
3. stores the final world/object position in `position`
4. projects it normally

This is the same kind of “turn a primitive into a scene object” step we used for the water plane in earlier parts.

---

## Sphere fragment shader

```glsl
varying vec3 position;

void main() {
  vec3 color = getSphereColor(position);

  vec4 info = texture2D(water, position.xz * 0.5 + 0.5);

  if (position.y < info.r)
    color *= underwaterColor * 1.2;

  gl_FragColor = vec4(color, 1.0);
}
```

---

## What the sphere fragment shader does

This shader shades the sphere directly, then checks whether the current fragment is below the water surface.

### Step 1 — Shade the sphere

```glsl
vec3 color = getSphereColor(position);
```

We reuse the sphere lighting helper from the shared GLSL block.

### Step 2 — Sample the water height above this fragment

```glsl
vec4 info = texture2D(water, position.xz * 0.5 + 0.5);
```

This maps the sphere fragment’s xz position into water-texture space and reads the water height.

### Step 3 — Compare the fragment y value to the water height

```glsl
if (position.y < info.r)
```

If the fragment lies below the water surface, then it is underwater.

### Step 4 — Apply underwater tint

```glsl
color *= underwaterColor * 1.2;
```

So the sphere becomes visually different when submerged.

This works because the sphere and the water surface both read the same water simulation texture.

That shared lookup is what keeps the waterline consistent.

---

# Step 10 — Add the Pool Shader

The pool also becomes a directly rendered object, so it needs its own shader pair.

---

## Pool vertex shader

```glsl
varying vec3 position;

void main() {
  position = gl_Vertex.xyz;
  position.y = ((1.0 - position.y) * (7.0 / 12.0) - 1.0) * poolHeight;
  gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);
}
```

---

## What the pool vertex shader does

It starts from the open-top cube mesh and reshapes it vertically into pool dimensions.

### Why remap `position.y`?

A unit cube starts with `gl_Vertex.y` in the range `[-1, 1]`, but those coordinates are not yet the pool shape we want. The pool is an open-top box whose rim should sit at the water level, with the walls and floor extending downward from there.

So instead of using the cube’s original y coordinates directly, the shader remaps them:

```glsl
position.y = ((1.0 - position.y) * (7.0 / 12.0) - 1.0) * poolHeight;
```

This does four things in sequence:

1. `1.0 - position.y` flips the original y direction so the top of the cube becomes depth `0`
2. `* (7.0 / 12.0)` scales that depth range
3. `- 1.0` shifts the whole shape downward into the scene
4. `* poolHeight` scales the final result

If you plug in the endpoints of the original cube:

- at `position.y = 1`, the remapped value becomes `-poolHeight`
- at `position.y = -1`, the remapped value becomes `poolHeight / 6`

So the pool mesh ends up spanning this vertical range:

```text
top rim    = -poolHeight
bottom     =  poolHeight / 6
```

That means the pool is not centered vertically. The top rim is placed lower in the scene, and the bottom extends only partway upward from there. Those constants are not meant to produce a mathematically elegant interval; they are hand-tuned scene-layout values chosen so the pool sits in a visually useful place relative to the water plane, sphere, and camera.

One subtle point is that this remapped mesh does not exactly match the box used in the water shader’s `traceSceneColor()` intersection test. The water shader traces against a simplified proxy box, while the pool vertex shader reshapes the rendered cube into the final visible pool. So the reflected and refracted rays are intersecting an approximation of the pool volume, not the literal rendered mesh. In practice that is good enough visually, but it is worth noting that the ray-traced proxy and the rendered pool geometry are only approximately the same.

---

## Pool fragment shader

```glsl
varying vec3 position;

void main() {
  gl_FragColor = vec4(getWallColor(position), 1.0);
}
```

---

## What the pool fragment shader does

This shader is simple because most of the interesting logic already lives in `getWallColor(position)`:

- detect wall/floor
- compute normal
- sample tile texture
- apply diffuse lighting

So the pool shader is mostly a thin wrapper around the helper.

---

# Step 11 — Delta: `main.js` Draw Flow

In Part 3, the app rendered only the water surface.

## Part 3-style draw

```javascript
renderer.renderWater(water, cubemap);
```

In Part 4, the scene now contains three objects.

So we update the draw flow.

## Part 4 draw sequence

```javascript
gl.enable(gl.DEPTH_TEST);
gl.enable(gl.CULL_FACE);

renderer.renderWater(water, cubemap);

gl.cullFace(gl.BACK;)
renderer.renderCube(water);
renderer.renderSphere(water);

gl.disable(gl.DEPTH_TEST);
gl.disable(gl.CULL_FACE);
```

We pull enabling the depth test and face culling to the draw loop since the objects need to be culled too.

---

# Step 12 — Where the New Code Goes in `renderer.js`

The new renderer code fits into clear buckets.

## Inside the constructor

Add:

- sphere state
- `sphereMesh`
- `cubeMesh`
- `tileTexture`
- new GLSL helper functions
- sphere shader creation
- pool shader creation
- updated water shader creation with new uniforms

## Below the constructor

Add:

- `Renderer.prototype.renderCube = function(water) { ... }`
- `Renderer.prototype.renderSphere = function(water) { ... }`

Update:

- `Renderer.prototype.renderWater = function(water, sky) { ... }`

This is the same structure as Part 3:

- constructor creates meshes, textures, and shaders
- render methods bind resources and draw objects

Part 4 just increases the number of scene objects.

---

# Full Listings of the Modified Functions

Below are the full functions that are modified or newly added in this part.

---

## `Renderer` constructor

```javascript
function Renderer() {
  this.waterMesh = GL.Mesh.plane({ detail: 200 });

  this.sphereCenter = new GL.Vector(0, 0, 0);
  this.sphereRadius = 0.25;
  this.sphereMesh = GL.Mesh.sphere({ detail: 10 });

  this.cubeMesh = GL.Mesh.cube();
  this.cubeMesh.triangles.splice(4, 2);
  this.cubeMesh.compile();

  this.tileTexture = GL.Texture.fromImage(
    document.getElementById('tiles'),
    {
      minFilter: gl.LINEAR_MIPMAP_LINEAR,
      wrap: gl.REPEAT
    }
  );

  var helperFunctions = '\
    const float IOR_AIR = 1.0;\
    const float IOR_WATER = 1.333;\
    const float poolHeight = 1.0;\
    \
    uniform sampler2D water;\
    uniform samplerCube sky;\
    uniform sampler2D tiles;\
    uniform vec3 light;\
    uniform vec3 sphereCenter;\
    uniform float sphereRadius;\
    uniform vec3 underwaterColor;\
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
    float intersectSphere(vec3 origin, vec3 ray, vec3 center, float radius) {\
      vec3 oc = origin - center;\
      float a = dot(ray, ray);\
      float b = 2.0 * dot(oc, ray);\
      float c = dot(oc, oc) - radius * radius;\
      float d = b * b - 4.0 * a * c;\
      if (d < 0.0) return 1e6;\
      float t = (-b - sqrt(d)) / (2.0 * a);\
      return t > 0.0 ? t : 1e6;\
    }\
    \
    vec3 getSphereColor(vec3 point) {\
      vec3 color = vec3(0.5);\
      vec3 normal = (point - sphereCenter) / sphereRadius;\
      float diffuse = max(0.0, dot(normal, -light)) * 0.5;\
      color += diffuse;\
      return color;\
    }\
    \
    vec3 getWallColor(vec3 point) {\
      vec3 normal;\
      vec3 color;\
      if (abs(point.x) > 0.999) {\
        normal = vec3(-point.x, 0.0, 0.0);\
        color = texture2D(tiles, point.yz * 0.5 + vec2(1.0, 0.5)).rgb;\
      } else if (abs(point.z) > 0.999) {\
        normal = vec3(0.0, 0.0, -point.z);\
        color = texture2D(tiles, point.yx * 0.5 + vec2(1.0, 0.5)).rgb;\
      } else {\
        normal = vec3(0.0, 1.0, 0.0);\
        color = texture2D(tiles, point.xz * 0.5 + 0.5).rgb;\
      }\
      float diffuse = max(0.0, dot(normal, -light));\
      return color * (0.5 + diffuse * 0.5);\
    }\
    \
    vec3 traceSceneColor(vec3 origin, vec3 ray, vec3 waterTint) {\
      float tSphere = intersectSphere(origin, ray, sphereCenter, sphereRadius);\
      vec2 tCube = intersectBox(origin, ray, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));\
      if (tSphere < tCube.y) {\
        vec3 hit = origin + ray * tSphere;\
        return getSphereColor(hit) * waterTint;\
      }\
      if (ray.y < 0.0) {\
        vec3 hit = origin + ray * tCube.y;\
        return getWallColor(hit) * waterTint;\
      }\
      return textureCube(sky, ray).rgb;\
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
        for (int j = 0; j < 5; j++) {\
          coord += info.ba * 0.005;\
          info = texture2D(water, coord);\
        }\
        return vec3(info.b, sqrt(max(0.0, 1.0 - dot(info.ba, info.ba))), info.a);\
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
        vec3 reflectedColor = traceSceneColor(position, reflectedRay, underwaterColor);\
        vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));\
        gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);\
      ' : '\
        vec3 reflectedRay = reflect(incomingRay, normal);\
        vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);\
        float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));\
        vec3 reflectedColor = traceSceneColor(position, reflectedRay, vec3(1.0));\
        vec3 refractedColor = traceSceneColor(position, refractedRay, vec3(1.0));\
        gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);\
      ') + '\
      }\
    ');
  }

  this.sphereShader = new GL.Shader('\
    uniform vec3 sphereCenter;\
    uniform float sphereRadius;\
    varying vec3 position;\
    void main() {\
      position = sphereCenter + gl_Vertex.xyz * sphereRadius;\
      gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);\
    }\
  ', helperFunctions + '\
    varying vec3 position;\
    void main() {\
      vec3 color = getSphereColor(position);\
      vec4 info = texture2D(water, position.xz * 0.5 + 0.5);\
      if (position.y < info.r) color *= underwaterColor * 1.2;\
      gl_FragColor = vec4(color, 1.0);\
    }\
  ');

  this.cubeShader = new GL.Shader('\
    varying vec3 position;\
    uniform float poolHeight;\
    void main() {\
      position = gl_Vertex.xyz;\
      position.y = ((1.0 - position.y) * (7.0 / 12.0) - 1.0) * poolHeight;\
      gl_Position = gl_ModelViewProjectionMatrix * vec4(position, 1.0);\
    }\
  ', helperFunctions + '\
    varying vec3 position;\
    void main() {\
      gl_FragColor = vec4(getWallColor(position), 1.0);\
    }\
  ');
}
```

---

## `Renderer.prototype.renderWater`

```javascript
Renderer.prototype.renderWater = function(water, sky) {
  var tracer = new GL.Raytracer();

  water.textureA.bind(0);
  sky.bind(1);
  this.tileTexture.bind(2);

  for (var i = 0; i < 2; i++) {
    gl.cullFace(i ? gl.BACK : gl.FRONT);

    this.waterShaders[i].uniforms({
      water: 0,
      sky: 1,
      tiles: 2,
      eye: tracer.eye,
      light: [0.0, 1.0, 0.0],
      sphereCenter: this.sphereCenter,
      sphereRadius: this.sphereRadius,
      underwaterColor: [0.4, 0.9, 1.0]
    }).draw(this.waterMesh);
  }

};
```

Note that removed enabling and disabling of face culling here, since it's done in the draw loop.

---

## `Renderer.prototype.renderSphere`

```javascript
Renderer.prototype.renderSphere = function(water) {
  water.textureA.bind(0);
  this.sphereShader.uniforms({
    water: 0,
    tiles: 2,
    light: [0.0, 1.0, 0.0],
    sphereCenter: this.sphereCenter,
    sphereRadius: this.sphereRadius,
    underwaterColor: [0.4, 0.9, 1.0]
  }).draw(this.sphereMesh);
};
```

---

## `Renderer.prototype.renderCube`

```javascript
Renderer.prototype.renderCube = function(water) {
  water.textureA.bind(0);
  this.tileTexture.bind(2);
  this.cubeShader.uniforms({
    water: 0,
    tiles: 2,
    light: [0.0, 1.0, 0.0],
    sphereCenter: this.sphereCenter,
    sphereRadius: this.sphereRadius,
    underwaterColor: [0.4, 0.9, 1.0],
    poolHeight: 1.0
  }).draw(this.cubeMesh);
};
```

---

## `main.js` draw/update section

```javascript
function draw() {
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
  gl.loadIdentity();

  gl.translate(0, 0, -4);
  gl.rotate(-angleX, 1, 0, 0);
  gl.rotate(-angleY, 0, 1, 0);
  gl.translate(0, 0.5, 0);

  gl.enable(gl.DEPTH_TEST);
  gl.enable(gl.CULL_FACE);

  renderer.renderWater(water, cubemap);

  gl.cullFace(gl.BACK;)
  renderer.renderCube(water);
  renderer.renderSphere(water);

  gl.disable(gl.DEPTH_TEST);
  gl.disable(gl.CULL_FACE);
}
```

---

# Result of Part 4

At the end of this stage you should see:

- a tiled pool interior
- a sphere inside the water
- reflections and refractions that can hit both the pool and the sphere
- direct rendering of the pool and sphere in addition to the water surface

This is the stage where the scene stops looking like isolated water shading and starts looking like a real place.

---

# What Comes Next

In Part 5, we add one of the most striking effects in the demo:

**caustics**.

These are the moving focused-light patterns cast by the water onto the pool interior and sphere.

That will require another GPU pass derived from the water surface shape and normals.