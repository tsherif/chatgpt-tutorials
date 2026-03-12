# WebGL Water Tutorial — Part 4: Pool Geometry and the Sphere

In Part 3 the water surface began behaving like a real optical interface.

However, the reflections and refractions were still interacting mostly with an *imaginary* scene.

In this part we introduce the real scene geometry:

• the **pool walls and floor**
• the **sphere inside the pool**

These objects interact with the water surface through the same ray‑based shading logic introduced in Part 3.

At the end of Part 4 the scene finally becomes spatially coherent.

---

# Architecture of the Scene

All scene elements depend on the **same water simulation texture**.

```
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

Because all three materials read from the same texture:

• they agree on the water height
• they agree on the surface normals
• they react consistently when waves move

This shared state is the key architectural idea.

---

# Files Modified in This Part

```
renderer.js
main.js
```

Unchanged:

```
water.js
```

---

# Step 1 — Add Sphere State

The renderer must know where the sphere is located.

Add these fields inside the `Renderer` constructor.

```javascript
this.sphereCenter = new GL.Vector(0,0,0);
this.sphereRadius = 0.25;
```

These values will later be controlled by the application.

---

# Step 2 — Add the Scene Meshes

The renderer now needs two additional meshes.

```javascript
this.sphereMesh = GL.Mesh.sphere({ detail: 10 });

this.cubeMesh = GL.Mesh.cube();
this.cubeMesh.triangles.splice(4,2);
this.cubeMesh.compile();
```

Explanation:

• `sphereMesh` will render the floating ball
• `cubeMesh` becomes the pool interior

---

# Why Remove Two Cube Triangles?

A cube normally has six faces.

The pool must be **open at the top**.

```
 cube

remove top face

result → pool interior
```

Each cube face consists of two triangles.

So removing two triangles deletes the top.

---

# Step 3 — Tile Texture for Pool Walls

The pool interior uses a tile texture.

Add this inside the renderer constructor.

```javascript
this.tileTexture = GL.Texture.fromImage(
  document.getElementById("tiles"),
  {
    minFilter: gl.LINEAR_MIPMAP_LINEAR,
    wrap: gl.REPEAT
  }
);
```

This texture will be sampled by the pool shader.

---

# Step 4 — Ray Intersection with the Sphere

The water shader needs to know if a reflection or refraction ray hits the sphere.

We add a ray–sphere intersection function.

```glsl
float intersectSphere(
  vec3 origin,
  vec3 ray,
  vec3 center,
  float radius
)
{

  vec3 oc = origin - center;

  float a = dot(ray,ray);
  float b = 2.0 * dot(oc,ray);
  float c = dot(oc,oc) - radius*radius;

  float d = b*b - 4.0*a*c;

  if (d < 0.0) return 1e6;

  float t = (-b - sqrt(d))/(2.0*a);

  return t > 0.0 ? t : 1e6;

}
```

This solves the quadratic equation for a sphere intersection.

---

# Ray–Sphere Geometry

```
          ray
           │
           ▼
      ----( )----
           ▲
        sphere
```

If the discriminant is negative, the ray misses the sphere.

---

# Step 5 — Sphere Shading Function

We add a helper that returns the sphere's base color.

```glsl
vec3 getSphereColor(vec3 point)
{

  vec3 color = vec3(0.5);

  vec3 normal = (point - sphereCenter)/sphereRadius;

  float diffuse = max(0.0, dot(normal,-light))*0.5;

  color += diffuse;

  return color;

}
```

This is intentionally simple lighting.

The sphere is mainly present to interact with the water.

---

# Step 6 — Pool Wall Shading

The pool shader must determine which wall is hit.

```glsl
vec3 getWallColor(vec3 point)
{

  vec3 normal;
  vec3 color;

  if(abs(point.x) > 0.999)
  {
    normal = vec3(-point.x,0.0,0.0);
    color = texture2D(tiles,point.yz*0.5+vec2(1.0,0.5)).rgb;
  }
  else if(abs(point.z) > 0.999)
  {
    normal = vec3(0.0,0.0,-point.z);
    color = texture2D(tiles,point.yx*0.5+vec2(1.0,0.5)).rgb;
  }
  else
  {
    normal = vec3(0.0,1.0,0.0);
    color = texture2D(tiles,point.xz*0.5+0.5).rgb;
  }

  float diffuse = max(0.0,dot(normal,-light));

  return color*(0.5 + diffuse*0.5);

}
```

The shader determines which wall the point lies on and assigns a normal.

---

# Pool Coordinate Diagram

```
        z
        │
   wall │ wall
        │
 ───────┼─────── x
        │
       floor
```

The shader detects which boundary was hit.

---

# Step 7 — Sphere Vertex Shader

```glsl
varying vec3 position;

void main()
{

  position = sphereCenter + gl_Vertex.xyz*sphereRadius;

  gl_Position = gl_ModelViewProjectionMatrix*
                vec4(position,1.0);

}
```

Explanation:

• `gl_Vertex` is the unit sphere
• multiply by radius to scale
• add center to translate

---

# Step 8 — Sphere Fragment Shader

```glsl
varying vec3 position;

void main()
{

  vec3 color = getSphereColor(position);

  vec4 info = texture2D(water,position.xz*0.5+0.5);

  if(position.y < info.r)
    color *= underwaterColor*1.2;

  gl_FragColor = vec4(color,1.0);

}
```

The sphere checks whether the fragment lies below the water surface.

If so it receives underwater tinting.

---

# Water Height Test

```
sphere fragment
     │
     ▼
compare y
with water height
```

Because the sphere samples the same water texture as the surface, both agree on where the water is.

---

# Step 9 — Pool Vertex Shader

The cube mesh must be reshaped into the pool dimensions.

```glsl
varying vec3 position;

void main()
{

  position = gl_Vertex.xyz;

  position.y = ((1.0-position.y)*(7.0/12.0)-1.0)*poolHeight;

  gl_Position = gl_ModelViewProjectionMatrix*
                vec4(position,1.0);

}
```

This remaps the cube vertically to form the pool interior.

---

# Step 10 — Rendering Order

The renderer now draws three objects.

```
1 pool
2 water
3 sphere
```

Implementation in `draw()`:

```javascript
renderer.sphereCenter = center;
renderer.sphereRadius = radius;

renderer.renderCube(water);
renderer.renderWater(water,cubemap);
renderer.renderSphere(water);
```

Depth testing ensures proper visibility.

---

# Ray Interaction Diagram

After this part, reflection and refraction rays can hit real objects.

```
view ray hits water
        │
        ├── reflected → sky or pool edge
        │
        └── refracted → sphere or floor
```

This is what finally anchors the water visually.

---

# Result of Part 4

At this stage the scene contains:

• textured pool walls
• pool floor
• sphere inside the water
• reflections and refractions interacting with both

The demo now resembles the final visual structure.

---

# What Comes Next

In **Part 5** we add one of the most striking effects:

**caustics**.

These are the moving light patterns produced when the water surface focuses sunlight onto the pool interior.

They will be computed using another GPU pass derived from the water surface normals.

