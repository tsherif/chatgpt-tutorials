# WebGL Water Tutorial — Part 5: Caustics

In Part 4 we introduced the **pool geometry and sphere**, creating a coherent scene that reflection and refraction rays could interact with.

In this part we add one of the most recognizable visual effects of water:

**caustics**.

Caustics are the moving light patterns produced when a wavy water surface focuses sunlight onto surfaces below it.

At the end of this part you will have:

• animated caustic light patterns on the pool floor
• caustics projected onto the sphere
• a new GPU pass that computes light focusing from the water surface

---

# What Are Caustics?

When light passes through a curved surface it bends according to Snell's Law.

A wavy water surface behaves like a constantly changing lens.

Some regions concentrate light, while others spread it out.

Where many light rays converge, the surface becomes brighter.

```
Sunlight
   │
   ▼
 ~~~~~~~ water surface
  \\ / /  refracted rays
   \\/
    \/
  bright spots
```

These bright regions are **caustics**.

---

# Why Real Caustics Are Expensive

Physically accurate caustics require tracing many light rays.

```
light source
   │
 thousands of rays
   │
   ▼
 surfaces
```

This is computationally expensive.

Instead the demo uses a clever approximation based on the **water surface normals**.

---

# Key Insight

The water simulation already gives us two critical pieces of data:

• the **surface position**
• the **surface normal**

From the surface normal we can compute the direction sunlight refracts through the water.

```
refracted = refract(-lightDir, normal, IOR_AIR / IOR_WATER)
```

If the refracted ray points strongly downward, that region focuses light onto the pool floor.

So the caustic brightness can be approximated by measuring the **downward component of the refracted ray**.

---

# Architecture of the Caustics Pass

The caustics computation runs as a separate fullscreen pass.

```
water texture
(height + normals)
        │
        ▼
caustics shader
        │
        ▼
caustics texture
        │
        ▼
pool + sphere shading
```

This texture stores the brightness of focused light.

---

# Step 1 — Add the Caustics Texture

Inside the `Renderer` constructor add:

```javascript
this.causticsTexture = new GL.Texture(
  1024,
  1024,
  { format: gl.RGB }
);
```

This texture will store the light intensity pattern produced by the water surface.

A relatively high resolution is used because caustic patterns contain fine details.

---

# Step 2 — Caustics Vertex Shader

The caustics pass uses a fullscreen quad.

```
varying vec2 coord;

void main() {

  coord = gl_Vertex.xy * 0.5 + 0.5;

  gl_Position = vec4(gl_Vertex,1.0);

}
```

Explanation:

```
gl_Vertex
```

contains the quad coordinates in clip space.

The shader converts them to UV coordinates for sampling the water texture.

---

# Step 3 — Caustics Fragment Shader

```glsl
uniform sampler2D water;
uniform vec3 light;

varying vec2 coord;

void main()
{

  vec4 info = texture2D(water,coord);

  vec3 normal = vec3(
    info.b,
    sqrt(1.0 - dot(info.ba,info.ba)),
    info.a
  );

  vec3 refracted = refract(
    -light,
    normal,
    1.0/1.333
  );

  float intensity = pow(
    max(0.0, dot(refracted, vec3(0.0,-1.0,0.0))),
    3.0
  );

  gl_FragColor = vec4(vec3(intensity),1.0);

}
```

---

# Shader Walkthrough

### 1. Sample water state

```
vec4 info = texture2D(water,coord);
```

This gives us the water surface normal components.

---

### 2. Reconstruct the normal

```
vec3 normal
```

The normal controls how light bends through the surface.

---

### 3. Compute refracted light direction

```
refract(-light, normal, 1.0/1.333)
```

This simulates sunlight entering the water.

---

### 4. Estimate focusing strength

```
dot(refracted, downward)
```

If the ray points strongly downward, light is concentrated.

The power function exaggerates the effect.

---

# Caustic Light Diagram

```
        light
         │
         ▼
      water
   /   /  \
  /   /    \
 focused rays

   bright caustic
```

The shader approximates this focusing behavior.

---

# Step 4 — Renderer Method

Add a function to compute caustics each frame.

```javascript
Renderer.prototype.updateCaustics = function(water)
{

  var self = this;

  this.causticsTexture.drawTo(function(){

    water.textureA.bind();

    self.causticsShader.uniforms({
      water:0,
      light:self.lightDir
    }).draw(GL.Mesh.plane());

  });

};
```

This renders the caustics texture using the water surface data.

---

# Step 5 — Use Caustics in Pool Shader

Inside `getWallColor()` add:

```glsl
vec3 caustic = texture2D(
  causticTex,
  point.xz * 0.5 + 0.5
).rgb;

color += caustic * 0.5;
```

This projects the caustic pattern onto the pool surfaces.

---

# Step 6 — Caustics on the Sphere

Inside `getSphereColor()` add:

```glsl
vec3 caustic = texture2D(
  causticTex,
  point.xz * 0.5 + 0.5
).rgb;

color += caustic;
```

Now the sphere receives animated light patterns.

---

# Step 7 — Update the Frame Loop

In `animate()` call the caustics update.

```javascript
renderer.updateCaustics(water);
```

Place this before drawing the scene.

---

# Full Frame Pipeline

After adding caustics the frame loop looks like this:

```
water simulation
      │
      ▼
update normals
      │
      ▼
compute caustics
      │
      ▼
render pool
render water
render sphere
```

---

# Result of Part 5

At this stage you should see:

• animated light patterns on the pool floor
• caustics on the sphere
• patterns that move with the water waves

This dramatically increases the realism of the scene.

---

# Why This Works So Well

Even though the shader does not trace real photons, the approximation captures the most visually important effect:

**areas where refracted rays converge become brighter.**

Because the water surface is constantly changing, the caustic texture evolves every frame.

---

# What Comes Next

In **Part 6** we add interaction and physical coupling between objects and water:

• mouse interaction
• sphere displacement generating waves
• ripple forces applied to the simulation

At that point the demo becomes fully interactive.

