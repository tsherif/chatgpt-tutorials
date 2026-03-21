# WebGL Water Tutorial — Part 5: Caustics

In Part 4 we introduced the pool geometry and sphere, creating a scene that reflection and refraction rays could interact with. The pool walls and floor were shaded with a tile texture and simple diffuse lighting, but they looked flat and static — there was no sense that light was bending through the water surface above.

In this part we add **caustics**: the animated bright patterns that appear on the pool floor and walls when sunlight focuses through a wavy water surface.

At the end of Part 5 you will have:

- a caustic texture computed each frame from the water surface
- animated light patterns on the pool walls and floor
- caustics projected onto the sphere
- an understanding of the area-ratio technique that makes this efficient

---

# What Are Caustics?

When light passes through a curved transparent surface, it bends according to Snell's Law. A wavy water surface acts like a constantly shifting lens. Some regions of the surface are concave and focus rays together; others are convex and spread rays apart.

Where many rays converge on the pool floor, the surface is brighter than average. Where rays diverge, it is darker. The animated pattern of bright and dark patches that results is what we call caustics.
```
          sunlight (parallel rays)
         │  │  │  │  │  │  │  │
         │  │  │  │  │  │  │  │
      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~  water surface
        \\ \\ │  │ / / \\ \\ │  │ / /       refracted rays
         \\  \│ /   \\ \\  \│ /
          \\  X       \\  X              convergence
     ─────────────────────────────────   pool floor
              bright       bright
```

---

# Why Real Caustics Are Expensive

Physically accurate caustics require tracing millions of light rays from the source, through the water surface, to the pool floor. The cost scales with the number of rays needed to produce a smooth result.

This demo uses a much cheaper approximation that runs entirely on the GPU and produces convincing results. The key insight is that we already have everything we need: the water simulation texture contains the surface height and normal at every point.

---

# The Area-Ratio Technique

The core idea is this:

> If a patch of water surface focuses light onto a smaller area of the pool floor, that area receives proportionally more light.

We can measure this directly. For each small triangle of the water mesh, we know where sunlight would hit the pool floor both:

- **without** refraction (as if the water surface were flat), and
- **with** refraction (using the actual displaced surface and its normal)

We use this information to project each triangle from the water mesh to the pool floor using both the refracted and flat-water rays. If the refracted triangle has a smaller area than the flat-water triangle, light has been concentrated — that region of the pool floor is brighter. If it has a larger area, light has been spread out — that region is darker.

The ratio of the two areas gives us the caustic brightness directly:
```
caustic brightness = flat area / refracted area
```

This is computed per-triangle on the GPU, which makes it extremely fast.

---

# The OES_standard_derivatives Extension

To compute triangle areas in the fragment shader, we need the `OES_standard_derivatives` extension.

Normally, a fragment shader only knows about the current fragment — it has no direct access to neighboring fragments. But screen-space derivatives give us a way around this.

For any value `v` computed in the fragment shader, the GPU can provide:

- `dFdx(v)` — how much `v` changes between this fragment and the fragment one step to the right
- `dFdy(v)` — how much `v` changes between this fragment and the fragment one step upward

These are computed by the GPU by comparing the value across a 2×2 block of fragments that are processed together. This is why they are sometimes called "quad derivatives". The GPU always processes fragments in groups, which makes finite differences between neighbors cheap (the GLSL spec defines this in terms of 2×2 blocks, though the exact hardware implementation varies). 

In the caustics shader, we pass the 3D world-space position of each vertex through as a varying. In the fragment shader, `dFdx(position)` and `dFdy(position)` give two edge vectors of the projected triangle in world space. The cross product of those two vectors gives the triangle's area via:
```
area = length(dFdx(position)) * length(dFdy(position))
```

This is not exact — it is a first-order approximation — but it is accurate enough for a visual effect and runs at full GPU speed with no readbacks or CPU involvement.

The extension must be explicitly enabled at the top of the fragment shader source:
```glsl
#extension GL_OES_standard_derivatives : enable
```

If the extension is not available, the demo falls back to a fixed brightness value rather than failing entirely.

---

# Architecture of the Caustics Pass

The caustics pass runs once per frame, before the pool and sphere are rendered. It produces a texture that the pool and sphere shaders then sample.
```
water simulation texture
(height + normals)
        │
        ▼
caustics vertex shader
  - sample water height and normal at each vertex
  - compute where flat-light ray hits pool floor (oldPos)
  - compute where refracted ray hits pool floor (newPos)
  - output screen position based on newPos
        │
        ▼
caustics fragment shader
  - compute oldPos triangle area via dFdx/dFdy
  - compute newPos triangle area via dFdx/dFdy
  - caustic brightness = oldArea / newArea
  - also compute sphere shadow and pool rim shadow
        │
        ▼
causticTex (1024×1024)
        │
  ┌─────┴─────┐
  ▼           ▼
pool        sphere
shader      shader
```

One important detail: the caustics pass draws the **same water mesh** used for rendering, not a fullscreen quad. Each vertex of the water mesh is projected to where its refracted light ray lands on the pool floor. This is what allows the area-ratio technique to work — we are literally rasterizing the refracted mesh onto the caustic texture.

---

# What Changes in Part 5

`water.js` is unchanged.

`main.js` gets one new call in the `update()` function.

`renderer.js` gets:

- a `causticTex` texture allocated in the constructor
- a `causticsShader` built in the constructor
- the `updateCaustics()` method added below the constructor
- `causticTex` bound and passed as a uniform in `renderWater()`, `renderSphere()`, and `renderCube()`
- `getWallColor()` and `getSphereColor()` in `helperFunctions` updated to sample and apply the caustic texture

---

# Step 1 — Allocate the Caustic Texture

Inside the `Renderer` constructor, add:
```javascript
this.causticTex = new GL.Texture(1024, 1024);
```

This is a standard 8-bit RGBA texture at 1024×1024. Unlike the water simulation textures, caustics do not need floating-point precision — 8-bit is sufficient for a light intensity pattern.

The texture is square and power-of-two because we will sample it with `REPEAT` wrap mode later.

---

# Step 2 — Update the helperFunctions Block

The `helperFunctions` string is the shared GLSL block prepended to every shader in the renderer. Two things need to change here.

First, add `causticTex` to the uniform declarations at the top of the block:

## Delta: uniform declarations in helperFunctions

### Part 4
```glsl
uniform vec3 light;
uniform vec3 sphereCenter;
uniform float sphereRadius;
uniform sampler2D tiles;
uniform sampler2D water;
```

### Part 5
```glsl
uniform vec3 light;
uniform vec3 sphereCenter;
uniform float sphereRadius;
uniform sampler2D tiles;
uniform sampler2D causticTex;
uniform sampler2D water;
```

Second, update `getSphereColor()` and `getWallColor()` to sample and apply the caustic texture. These are the most significant changes in Part 5. The full updated versions of both functions are shown below with explanation.

---

## Updated `getSphereColor()`

### Part 4
```glsl
vec3 getSphereColor(vec3 point) {
  vec3 color = vec3(0.5);
  vec3 normal = (point - sphereCenter) / sphereRadius;
  float diffuse = max(0.0, dot(normal, -light)) * 0.5;
  color += diffuse;
  return color;
}
```

### Part 5
```glsl
vec3 getSphereColor(vec3 point) {
  vec3 color = vec3(0.5);

  /* ambient occlusion with walls */
  color *= 1.0 - 0.9 / pow((1.0 + sphereRadius - abs(point.x)) / sphereRadius, 3.0);
  color *= 1.0 - 0.9 / pow((1.0 + sphereRadius - abs(point.z)) / sphereRadius, 3.0);
  color *= 1.0 - 0.9 / pow((point.y + 1.0 + sphereRadius) / sphereRadius, 3.0);

  /* caustics */
  vec3 sphereNormal = (point - sphereCenter) / sphereRadius;
  vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);
  float diffuse = max(0.0, dot(-refractedLight, sphereNormal)) * 0.5;
  vec4 info = texture2D(water, point.xz * 0.5 + 0.5);
  if (point.y < info.r) {
    vec4 caustic = texture2D(causticTex, 0.75 * (point.xz - point.y * refractedLight.xz / refractedLight.y) * 0.5 + 0.5);
    diffuse *= caustic.r * 4.0;
  }
  color += diffuse;

  return color;
}
```

### What changed and why

**Ambient occlusion** is new. The three `color *=` lines darken the sphere as it approaches the pool walls and floor. Each line computes a proximity factor for one boundary — the x walls, the z walls, and the floor — and multiplies it into the color. This is analytic ambient occlusion: it uses the geometry of the scene to estimate how much ambient light would be blocked, without any additional render passes.

Each line darkens the sphere based on its proximity to one of the pool boundaries — the x walls, the z walls, and the floor.

Take the first line as an example:
```glsl
color *= 1.0 - 0.9 / pow((1.0 + sphereRadius - abs(point.x)) / sphereRadius, 3.0);
```

The pool walls sit at `x = ±1`. `abs(point.x)` is the distance of the sphere surface point from the center along x, so `1.0 - abs(point.x)` is how far that point is from the nearest x wall. Adding `sphereRadius` gives a measure of how close the sphere as a whole is to that wall — when the sphere is just touching the wall, `1.0 + sphereRadius - abs(point.x)` equals `sphereRadius` at the contact point, and larger than `sphereRadius` everywhere else.

Dividing by `sphereRadius` normalizes this so that the value is `1.0` at the contact point and grows larger as the sphere moves away. Call this normalized distance `d`.

The darkening factor is then:
```
1.0 - 0.9 / pow(d, 3.0)
```

When `d = 1` (sphere touching the wall), this is `1.0 - 0.9 = 0.1` — the sphere is darkened to 10% brightness at that point. As `d` grows larger, `pow(d, 3.0)` grows quickly, `0.9 / pow(d, 3.0)` shrinks toward zero, and the factor approaches `1.0` — no darkening. The cubic exponent makes the falloff steep, so the darkening is concentrated close to the wall rather than spread broadly across the sphere.

The third line works the same way but for the floor at `y = -1`:
```glsl
color *= 1.0 - 0.9 / pow((point.y + 1.0 + sphereRadius) / sphereRadius, 3.0);
```

`point.y + 1.0` is the height of the surface point above the floor. Adding `sphereRadius` and dividing by `sphereRadius` gives the same normalized distance `d`, and the same darkening function applies.

So the three lines together darken the sphere wherever it is close to any pool boundary, approximating the way ambient light would be occluded when the sphere is near a wall or the floor.

**Caustic sampling** replaces the simple diffuse term. Instead of computing diffuse from the light direction directly, the shader:

1. Computes `refractedLight` — the direction sunlight takes after refracting through a flat water surface (not the actual wavy surface; this is a fixed reference direction used to project caustics onto the geometry).
2. Checks whether this sphere fragment is below the water surface by comparing `point.y` against the stored water height `info.r`.
3. If underwater, samples `causticTex` at the position where the refracted light ray from this point would intersect the water surface. The expression `point.xz - point.y * refractedLight.xz / refractedLight.y` projects the surface point back up to the waterline along the refracted light direction. The `0.75` scale factor is the same one used when generating the caustic texture — both sides of the lookup must use the same projection.
4. Multiplies `diffuse` by `caustic.r * 4.0` to modulate the lighting. The `4.0` compensates for the relatively low average value in the caustic texture.

---

## Updated `getWallColor()`

### Part 4
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

### Part 5
```glsl
vec3 getWallColor(vec3 point) {
  float scale = 0.5;

  vec3 wallColor;
  vec3 normal;
  if (abs(point.x) > 0.999) {
    wallColor = texture2D(tiles, point.yz * 0.5 + vec2(1.0, 0.5)).rgb;
    normal = vec3(-point.x, 0.0, 0.0);
  } else if (abs(point.z) > 0.999) {
    wallColor = texture2D(tiles, point.yx * 0.5 + vec2(1.0, 0.5)).rgb;
    normal = vec3(0.0, 0.0, -point.z);
  } else {
    wallColor = texture2D(tiles, point.xz * 0.5 + 0.5).rgb;
    normal = vec3(0.0, 1.0, 0.0);
  }

  scale /= length(point); /* pool ambient occlusion */
  scale *= 1.0 - 0.9 / pow(length(point - sphereCenter) / sphereRadius, 4.0); /* sphere ambient occlusion */

  /* caustics */
  vec3 refractedLight = -refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);
  float diffuse = max(0.0, dot(refractedLight, normal));
  vec4 info = texture2D(water, point.xz * 0.5 + 0.5);
  if (point.y < info.r) {
    vec4 caustic = texture2D(causticTex, 0.75 * (point.xz - point.y * refractedLight.xz / refractedLight.y) * 0.5 + 0.5);
    scale += diffuse * caustic.r * 2.0 * caustic.g;
  } else {
    /* shadow for the rim of the pool */
    vec2 t = intersectCube(point, refractedLight, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
    diffuse *= 1.0 / (1.0 + exp(-200.0 / (1.0 + 10.0 * (t.y - t.x)) * (point.y + refractedLight.y * t.y - 2.0 / 12.0)));
    scale += diffuse * 0.5;
  }

  return wallColor * scale;
}
```

### What changed and why

**The lighting model is restructured** around a `scale` value that accumulates all lighting contributions multiplicatively, rather than computing a single diffuse term at the end.

**Pool ambient occlusion** (`scale /= length(point)`) darkens wall and floor fragments that are closer to the center of the pool. Fragments near the corners and edges are further from the origin, so they receive a smaller darkening factor. This is a crude but effective approximation of how light bouncing around inside the pool would be attenuated near its interior.

**Sphere ambient occlusion** (`scale *= 1.0 - 0.9 / pow(...)`) darkens wall and floor fragments that are close to the sphere. This simulates the sphere blocking ambient light from reaching those nearby surfaces.

**Caustic sampling** follows the same projection logic as in `getSphereColor()`, but uses two channels from the caustic texture:
```glsl
scale += diffuse * caustic.r * 2.0 * caustic.g;
```

`caustic.r` is the brightness ratio computed in the caustics vertex shader. `caustic.g` is a sphere shadow mask also computed there — it is close to 0 where the sphere is blocking the light and 1 elsewhere. Multiplying by both means caustics are darkened where the sphere casts a shadow, which produces the correct soft shadow underneath the sphere.

**Above-water surfaces** use a different path. When `point.y >= info.r`, the wall or floor fragment is above the waterline and should not receive caustic lighting. Instead it receives a soft shadow from the pool rim — a logistic function that fades lighting out at the point where the refracted ray would emerge from the pool. This prevents the top edges of the pool walls from being lit as if they were underwater.

---

# Step 3 — Build the Caustics Shader

The caustics shader is the most complex shader in the entire demo. It is built inside the `Renderer` constructor after all the other shaders. Add the following:
```javascript
var hasDerivatives = !!gl.getExtension('OES_standard_derivatives');
this.causticsShader = new GL.Shader(helperFunctions + '\
  varying vec3 oldPos;\
  varying vec3 newPos;\
  varying vec3 ray;\
  \
  /* project the ray onto the plane */\
  vec3 project(vec3 origin, vec3 ray, vec3 refractedLight) {\
    vec2 tcube = intersectCube(origin, ray, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));\
    origin += ray * tcube.y;\
    float tplane = (-origin.y - 1.0) / refractedLight.y;\
    return origin + refractedLight * tplane;\
  }\
  \
  void main() {\
    vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);\
    info.ba *= 0.5;\
    vec3 normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);\
    \
    /* project the vertices along the refracted vertex ray */\
    vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);\
    ray = refract(-light, normal, IOR_AIR / IOR_WATER);\
    oldPos = project(gl_Vertex.xzy, refractedLight, refractedLight);\
    newPos = project(gl_Vertex.xzy + vec3(0.0, info.r, 0.0), ray, refractedLight);\
    \
    gl_Position = vec4(0.75 * (newPos.xz + refractedLight.xz / refractedLight.y), 0.0, 1.0);\
  }\
', (hasDerivatives ? '#extension GL_OES_standard_derivatives : enable\n' : '') + '\
  ' + helperFunctions + '\
  varying vec3 oldPos;\
  varying vec3 newPos;\
  varying vec3 ray;\
  \
  void main() {\
    ' + (hasDerivatives ? '\
      /* if the triangle gets smaller, it gets brighter, and vice versa */\
      float oldArea = length(dFdx(oldPos)) * length(dFdy(oldPos));\
      float newArea = length(dFdx(newPos)) * length(dFdy(newPos));\
      gl_FragColor = vec4(oldArea / newArea * 0.2, 1.0, 0.0, 0.0);\
    ' : '\
      gl_FragColor = vec4(0.2, 0.2, 0.0, 0.0);\
    ') + '\
    \
    vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);\
    \
    /* compute a blob shadow and make sure we only draw a shadow if the player is blocking the light */\
    vec3 dir = (sphereCenter - newPos) / sphereRadius;\
    vec3 area = cross(dir, refractedLight);\
    float shadow = dot(area, area);\
    float dist = dot(dir, -refractedLight);\
    shadow = 1.0 + (shadow - 1.0) / (0.05 + dist * 0.025);\
    shadow = clamp(1.0 / (1.0 + exp(-shadow)), 0.0, 1.0);\
    shadow = mix(1.0, shadow, clamp(dist * 2.0, 0.0, 1.0));\
    gl_FragColor.g = shadow;\
    \
    /* shadow for the rim of the pool */\
    vec2 t = intersectCube(newPos, -refractedLight, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));\
    gl_FragColor.r *= 1.0 / (1.0 + exp(-200.0 / (1.0 + 10.0 * (t.y - t.x)) * (newPos.y - refractedLight.y * t.y - 2.0 / 12.0)));\
  }\
');
```

---

# Understanding the Caustics Vertex Shader

The vertex shader is where most of the work happens. It runs once per vertex of the water mesh.

## Reading the water surface
```glsl
vec4 info = texture2D(water, gl_Vertex.xy * 0.5 + 0.5);
info.ba *= 0.5;
vec3 normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
```

This samples the water simulation texture and reconstructs the surface normal, the same as in the water renderer. The `info.ba *= 0.5` halves the normal's x and z components before reconstruction. This reduces the steepness of the normals used for caustic projection, which produces softer and more visually plausible caustic patterns. Without it, the caustics would be overly sharp and contrasty.

## The flat-water reference direction
```glsl
vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);
```

This computes where sunlight would travel after refracting through a **perfectly flat** water surface — that is, a surface with normal `(0, 1, 0)`. This reference direction is used in two places: to project `oldPos` (the unperturbed position on the pool floor), and as a common reference frame for the caustic texture coordinates used in `getWallColor()` and `getSphereColor()`.

## The actual refracted ray
```glsl
ray = refract(-light, normal, IOR_AIR / IOR_WATER);
```

This computes where sunlight actually travels after refracting through the real displaced surface at this vertex, using the actual wavy normal. This is the ray that carries focused or defocused light.

## Projecting both positions to the pool floor
```glsl
oldPos = project(gl_Vertex.xzy, refractedLight, refractedLight);
newPos = project(gl_Vertex.xzy + vec3(0.0, info.r, 0.0), ray, refractedLight);
```

The `project()` helper traces a ray from a given origin until it exits the pool box, then continues along `refractedLight` until it hits the pool floor (`y = -1`).

- `oldPos` starts from the flat grid position (no height displacement) and travels along `refractedLight`. This gives where the light would land if the water were flat.
- `newPos` starts from the actual displaced surface position (`gl_Vertex.xzy + vec3(0.0, info.r, 0.0)`) and travels along the actual refracted ray `ray`, but then follows `refractedLight` for the final step to the floor. The shared final direction keeps the projection consistent with the sampling coordinates used in `getWallColor()`.

## Outputting the screen position
```glsl
gl_Position = vec4(0.75 * (newPos.xz + refractedLight.xz / refractedLight.y), 0.0, 1.0);
```

This is the key step that makes the whole technique work. Instead of projecting to screen space using the camera, we project the vertex to the **caustic texture** using `newPos` — where the refracted ray lands on the pool floor.

The caustic texture is parameterized in the XZ plane. So the screen position in the caustic rendering pass corresponds directly to a position on the pool floor. Triangles that have been compressed by light focusing will rasterize into fewer fragments, which is exactly what the area ratio captures.

The `0.75` scale factor maps the pool floor coordinates (which run from -1 to 1) into texture space. This same factor appears in the `getWallColor()` and `getSphereColor()` texture lookups, ensuring the caustic texture coordinates are consistent everywhere.

---

# Understanding the Caustics Fragment Shader

## The area ratio
```glsl
float oldArea = length(dFdx(oldPos)) * length(dFdy(oldPos));
float newArea = length(dFdx(newPos)) * length(dFdy(newPos));
gl_FragColor = vec4(oldArea / newArea * 0.2, 1.0, 0.0, 0.0);
```

`dFdx(oldPos)` gives the rate of change of `oldPos` across a screen pixel — essentially one edge of the triangle in world-space coordinates. `dFdy(oldPos)` gives the other edge. Their lengths multiplied approximate the area of the triangle as it would be on the pool floor without refraction.

`newArea` is the same measurement for `newPos` — the actual refracted triangle area on the pool floor.

The ratio `oldArea / newArea` is the caustic brightness:

- If the refracted triangle is smaller than the flat-water triangle (light has been focused), the ratio is greater than 1 and the fragment is bright.
- If the refracted triangle is larger (light has been spread out), the ratio is less than 1 and the fragment is dim.

The `* 0.2` scale factor keeps the average brightness at a reasonable level.

This value is written to `gl_FragColor.r`. The initial value of `gl_FragColor.g` is set to `1.0` here, which is the default for the sphere shadow mask — meaning no shadow by default.

## Additive blending

The caustic texture is rendered with additive blending enabled. Each triangle of the water mesh contributes its brightness independently to the caustic texture. Where many triangles land in the same region of the texture — because light is being focused there — the contributions accumulate, producing the bright hotspots characteristic of caustics.

## Sphere shadow
```glsl
vec3 dir = (sphereCenter - newPos) / sphereRadius;
vec3 area = cross(dir, refractedLight);
float shadow = dot(area, area);
float dist = dot(dir, -refractedLight);
shadow = 1.0 + (shadow - 1.0) / (0.05 + dist * 0.025);
shadow = clamp(1.0 / (1.0 + exp(-shadow)), 0.0, 1.0);
shadow = mix(1.0, shadow, clamp(dist * 2.0, 0.0, 1.0));
gl_FragColor.g = shadow;
```

This computes an analytic soft shadow from the sphere. The idea is to check whether the refracted light ray from the pool floor back toward the light source passes close to the sphere.

`dir` is the vector from `newPos` (the floor point) to the sphere center, normalized by the sphere radius.

`cross(dir, refractedLight)` gives the perpendicular distance between the refracted light ray and the sphere center. `dot(area, area)` squares this distance.

The `dist = dot(dir, -refractedLight)` term measures how far the sphere center is along the light ray — used to fade the shadow as the sphere moves further from the floor point.

The logistic function `1.0 / (1.0 + exp(-shadow))` produces a smooth 0-to-1 transition for the shadow boundary. The result is stored in `gl_FragColor.g`, which `getWallColor()` and `getSphereColor()` multiply against the caustic brightness.

## Pool rim shadow
```glsl
vec2 t = intersectCube(newPos, -refractedLight, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
gl_FragColor.r *= 1.0 / (1.0 + exp(-200.0 / (1.0 + 10.0 * (t.y - t.x)) * (newPos.y - refractedLight.y * t.y - 2.0 / 12.0)));
```

This fades out caustic light for floor points that are near the waterline — the rim of the pool — where the light path back up through the water is very short and the water surface is not really above those points. This prevents the unrealistic band of caustic brightness that would otherwise appear along the top edge of the pool walls.

---

# Step 4 — Add the updateCaustics Method

Below the constructor, add:
```javascript
Renderer.prototype.updateCaustics = function(water) {
  if (!this.causticsShader) return;
  var this_ = this;
  this.causticTex.drawTo(function() {
    gl.clear(gl.COLOR_BUFFER_BIT);
    water.textureA.bind(0);
    this_.causticsShader.uniforms({
      light: this_.lightDir,
      water: 0,
      sphereCenter: this_.sphereCenter,
      sphereRadius: this_.sphereRadius
    }).draw(this_.waterMesh);
  });
};
```

This renders the caustic texture by drawing the water mesh into `causticTex` using `causticsShader`. The framebuffer is cleared at the start of each frame — caustics are recomputed entirely from the current water state each time.

Note that `causticTex` does not need the tile texture or sky cubemap. It only needs the water simulation texture and the sphere position. The pool and sphere shaders are responsible for combining the caustic brightness with the surface color.

---

# Step 5 — Delta: Pass causticTex into renderWater, renderSphere, renderCube

All three render methods need to bind `causticTex` and pass it as a uniform, since `getWallColor()` and `getSphereColor()` now reference it.

## renderWater

### Part 4
```javascript
Renderer.prototype.renderWater = function(water, sky) {
  var tracer = new GL.Raytracer();
  water.textureA.bind(0);
  this.tileTexture.bind(1);
  sky.bind(2);
  gl.enable(gl.CULL_FACE);
  for (var i = 0; i < 2; i++) {
    gl.cullFace(i ? gl.BACK : gl.FRONT);
    this.waterShaders[i].uniforms({
      light: this.lightDir,
      water: 0,
      tiles: 1,
      sky: 2,
      eye: tracer.eye,
      sphereCenter: this.sphereCenter,
      sphereRadius: this.sphereRadius,
      underwaterColor: [0.4, 0.9, 1.0]
    }).draw(this.waterMesh);
  }
  gl.disable(gl.CULL_FACE);
};
```

### Part 5
```javascript
Renderer.prototype.renderWater = function(water, sky) {
  var tracer = new GL.Raytracer();
  water.textureA.bind(0);
  this.tileTexture.bind(1);
  sky.bind(2);
  this.causticTex.bind(3);
  gl.enable(gl.CULL_FACE);
  for (var i = 0; i < 2; i++) {
    gl.cullFace(i ? gl.BACK : gl.FRONT);
    this.waterShaders[i].uniforms({
      light: this.lightDir,
      water: 0,
      tiles: 1,
      sky: 2,
      causticTex: 3,
      eye: tracer.eye,
      sphereCenter: this.sphereCenter,
      sphereRadius: this.sphereRadius
    }).draw(this.waterMesh);
  }
  gl.disable(gl.CULL_FACE);
};
```

Note that `underwaterColor` is removed from the uniforms here. In Part 4 it was passed explicitly, but in the reference code it is defined as a constant in `helperFunctions` rather than as a uniform, so it does not need to be passed.

## renderSphere

### Part 4
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

### Part 5
```javascript
Renderer.prototype.renderSphere = function() {
  water.textureA.bind(0);
  this.causticTex.bind(1);
  this.sphereShader.uniforms({
    light: this.lightDir,
    water: 0,
    causticTex: 1,
    sphereCenter: this.sphereCenter,
    sphereRadius: this.sphereRadius
  }).draw(this.sphereMesh);
};
```

The `water` argument is dropped from the signature because `renderSphere` accesses the global `water` variable directly, matching the reference code. `lightDir` is now taken from `this.lightDir` rather than a hardcoded vector.

## renderCube

### Part 4
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

### Part 5
```javascript
Renderer.prototype.renderCube = function() {
  gl.enable(gl.CULL_FACE);
  water.textureA.bind(0);
  this.tileTexture.bind(1);
  this.causticTex.bind(2);
  this.cubeShader.uniforms({
    light: this.lightDir,
    water: 0,
    tiles: 1,
    causticTex: 2,
    sphereCenter: this.sphereCenter,
    sphereRadius: this.sphereRadius
  }).draw(this.cubeMesh);
  gl.disable(gl.CULL_FACE);
};
```

Face culling moves into `renderCube` itself in the reference code. The `water` argument is dropped for the same reason as `renderSphere`.

---

# Step 6 — Delta: main.js

## Part 4
```javascript
function update(seconds) {
  if (seconds > 1) return;

  water.stepSimulation();
  water.stepSimulation();
  water.updateNormals();

  draw();
}
```

## Part 5
```javascript
function update(seconds) {
  if (seconds > 1) return;

  water.stepSimulation();
  water.stepSimulation();
  water.updateNormals();
  renderer.updateCaustics(water);

  draw();
}
```

`updateCaustics` is called after `updateNormals` because the caustics shader samples the water normal data stored in the simulation texture. The normals must be current before the caustic projection is computed.

---

# Full Frame Pipeline

After adding caustics, the per-frame sequence is:
```
stepSimulation()
stepSimulation()
updateNormals()
updateCaustics()
      │
      ▼
renderCube()      ← samples causticTex
renderWater()     ← samples causticTex (via getWallColor / getSphereColor in ray tracing)
renderSphere()    ← samples causticTex
```

---

# Full Listings of Modified Functions

## helperFunctions (complete)
```javascript
var helperFunctions = '\
  const float IOR_AIR = 1.0;\
  const float IOR_WATER = 1.333;\
  const vec3 abovewaterColor = vec3(0.25, 1.0, 1.25);\
  const vec3 underwaterColor = vec3(0.4, 0.9, 1.0);\
  const float poolHeight = 1.0;\
  uniform vec3 light;\
  uniform vec3 sphereCenter;\
  uniform float sphereRadius;\
  uniform sampler2D tiles;\
  uniform sampler2D causticTex;\
  uniform sampler2D water;\
  \
  vec2 intersectCube(vec3 origin, vec3 ray, vec3 cubeMin, vec3 cubeMax) {\
    vec3 tMin = (cubeMin - origin) / ray;\
    vec3 tMax = (cubeMax - origin) / ray;\
    vec3 t1 = min(tMin, tMax);\
    vec3 t2 = max(tMin, tMax);\
    float tNear = max(max(t1.x, t1.y), t1.z);\
    float tFar = min(min(t2.x, t2.y), t2.z);\
    return vec2(tNear, tFar);\
  }\
  \
  float intersectSphere(vec3 origin, vec3 ray, vec3 sphereCenter, float sphereRadius) {\
    vec3 toSphere = origin - sphereCenter;\
    float a = dot(ray, ray);\
    float b = 2.0 * dot(toSphere, ray);\
    float c = dot(toSphere, toSphere) - sphereRadius * sphereRadius;\
    float discriminant = b*b - 4.0*a*c;\
    if (discriminant > 0.0) {\
      float t = (-b - sqrt(discriminant)) / (2.0 * a);\
      if (t > 0.0) return t;\
    }\
    return 1.0e6;\
  }\
  \
  vec3 getSphereColor(vec3 point) {\
    vec3 color = vec3(0.5);\
    \
    /* ambient occlusion with walls */\
    color *= 1.0 - 0.9 / pow((1.0 + sphereRadius - abs(point.x)) / sphereRadius, 3.0);\
    color *= 1.0 - 0.9 / pow((1.0 + sphereRadius - abs(point.z)) / sphereRadius, 3.0);\
    color *= 1.0 - 0.9 / pow((point.y + 1.0 + sphereRadius) / sphereRadius, 3.0);\
    \
    /* caustics */\
    vec3 sphereNormal = (point - sphereCenter) / sphereRadius;\
    vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);\
    float diffuse = max(0.0, dot(-refractedLight, sphereNormal)) * 0.5;\
    vec4 info = texture2D(water, point.xz * 0.5 + 0.5);\
    if (point.y < info.r) {\
      vec4 caustic = texture2D(causticTex, 0.75 * (point.xz - point.y * refractedLight.xz / refractedLight.y) * 0.5 + 0.5);\
      diffuse *= caustic.r * 4.0;\
    }\
    color += diffuse;\
    \
    return color;\
  }\
  \
  vec3 getWallColor(vec3 point) {\
    float scale = 0.5;\
    \
    vec3 wallColor;\
    vec3 normal;\
    if (abs(point.x) > 0.999) {\
      wallColor = texture2D(tiles, point.yz * 0.5 + vec2(1.0, 0.5)).rgb;\
      normal = vec3(-point.x, 0.0, 0.0);\
    } else if (abs(point.z) > 0.999) {\
      wallColor = texture2D(tiles, point.yx * 0.5 + vec2(1.0, 0.5)).rgb;\
      normal = vec3(0.0, 0.0, -point.z);\
    } else {\
      wallColor = texture2D(tiles, point.xz * 0.5 + 0.5).rgb;\
      normal = vec3(0.0, 1.0, 0.0);\
    }\
    \
    scale /= length(point);\
    scale *= 1.0 - 0.9 / pow(length(point - sphereCenter) / sphereRadius, 4.0);\
    \
    /* caustics */\
    vec3 refractedLight = -refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);\
    float diffuse = max(0.0, dot(refractedLight, normal));\
    vec4 info = texture2D(water, point.xz * 0.5 + 0.5);\
    if (point.y < info.r) {\
      vec4 caustic = texture2D(causticTex, 0.75 * (point.xz - point.y * refractedLight.xz / refractedLight.y) * 0.5 + 0.5);\
      scale += diffuse * caustic.r * 2.0 * caustic.g;\
    } else {\
      vec2 t = intersectCube(point, refractedLight, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));\
      diffuse *= 1.0 / (1.0 + exp(-200.0 / (1.0 + 10.0 * (t.y - t.x)) * (point.y + refractedLight.y * t.y - 2.0 / 12.0)));\
      scale += diffuse * 0.5;\
    }\
    \
    return wallColor * scale;\
  }\
';
```

## Renderer.prototype.updateCaustics (complete)
```javascript
Renderer.prototype.updateCaustics = function(water) {
  if (!this.causticsShader) return;
  var this_ = this;
  this.causticTex.drawTo(function() {
    gl.clear(gl.COLOR_BUFFER_BIT);
    water.textureA.bind(0);
    this_.causticsShader.uniforms({
      light: this_.lightDir,
      water: 0,
      sphereCenter: this_.sphereCenter,
      sphereRadius: this_.sphereRadius
    }).draw(this_.waterMesh);
  });
};
```

## Renderer.prototype.renderWater (complete)
```javascript
Renderer.prototype.renderWater = function(water, sky) {
  var tracer = new GL.Raytracer();
  water.textureA.bind(0);
  this.tileTexture.bind(1);
  sky.bind(2);
  this.causticTex.bind(3);
  gl.enable(gl.CULL_FACE);
  for (var i = 0; i < 2; i++) {
    gl.cullFace(i ? gl.BACK : gl.FRONT);
    this.waterShaders[i].uniforms({
      light: this.lightDir,
      water: 0,
      tiles: 1,
      sky: 2,
      causticTex: 3,
      eye: tracer.eye,
      sphereCenter: this.sphereCenter,
      sphereRadius: this.sphereRadius
    }).draw(this.waterMesh);
  }
  gl.disable(gl.CULL_FACE);
};
```

## Renderer.prototype.renderSphere (complete)
```javascript
Renderer.prototype.renderSphere = function() {
  water.textureA.bind(0);
  this.causticTex.bind(1);
  this.sphereShader.uniforms({
    light: this.lightDir,
    water: 0,
    causticTex: 1,
    sphereCenter: this.sphereCenter,
    sphereRadius: this.sphereRadius
  }).draw(this.sphereMesh);
};
```

## Renderer.prototype.renderCube (complete)
```javascript
Renderer.prototype.renderCube = function() {
  gl.enable(gl.CULL_FACE);
  water.textureA.bind(0);
  this.tileTexture.bind(1);
  this.causticTex.bind(2);
  this.cubeShader.uniforms({
    light: this.lightDir,
    water: 0,
    tiles: 1,
    causticTex: 2,
    sphereCenter: this.sphereCenter,
    sphereRadius: this.sphereRadius
  }).draw(this.cubeMesh);
  gl.disable(gl.CULL_FACE);
};
```

---

# Result of Part 5

At the end of this stage you should see:

- animated bright caustic patterns on the pool floor and walls
- caustics on the sphere that move with the water
- a soft shadow underneath the sphere where it blocks the light
- the pool rim correctly shadowed above the waterline

---

# What Comes Next

In Part 6 we make the scene fully interactive.

That means adding:

- mouse input to create ripples on the water
- camera orbit on drag
- sphere dragging
- sphere physics including gravity and buoyancy
- the `moveSphere` pass in `water.js` that couples sphere movement to the water simulation