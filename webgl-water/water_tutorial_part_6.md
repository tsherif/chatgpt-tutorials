# WebGL Water Tutorial — Part 6: Interaction and Sphere–Water Coupling

In Part 5 we added **caustics**, giving the pool realistic animated lighting patterns.

In this final major step we make the scene **interactive**.

Two new behaviors are added:

• the user can create ripples by clicking the water
• the sphere displaces water and generates waves as it moves

This connects the water simulation with the rest of the scene and completes the demo.

---

# Overview of the Interaction System

The interaction system converts mouse input into disturbances in the water simulation.

```
mouse click
     │
     ▼
ray from camera
     │
     ▼
intersection with water surface
     │
     ▼
apply drop impulse
     │
     ▼
ripples propagate
```

The sphere uses the same mechanism to disturb the water automatically.

---

# Step 1 — Raycasting From the Camera

To determine where the user clicked on the water surface we cast a ray from the camera.

LightGL provides a helper:

```
GL.Raytracer
```

This class converts screen coordinates into world‑space rays.

---

# Mouse Event Handler

Inside `main.js` add a mouse handler.

```javascript
var tracer = new GL.Raytracer();

canvas.onmousedown = function(e) {

  var ray = tracer.getRayForPixel(e.x, e.y);

  var t = -tracer.eye.y / ray.y;

  var hit = tracer.eye.add(ray.multiply(t));

  water.addDrop(hit.x, hit.z, 0.03, 0.04);

};
```

Explanation:

1. Convert the screen pixel into a ray.
2. Intersect the ray with the water plane.
3. Convert the hit point to water coordinates.
4. Apply a disturbance.

---

# Ray–Plane Intersection

The intersection with the water surface uses a simple equation.

```
plane: y = 0

ray: origin + t * direction
```

Solve for `t`.

```
t = -origin.y / direction.y
```

The hit point is then

```
origin + t * direction
```

---

# Coordinate Conversion

The water simulation uses coordinates in the range

```
[-1 , 1]
```

The ray intersection produces world coordinates in the same range.

So the x and z values can be passed directly to the simulation.

---

# Step 2 — Mouse Drag Ripples

To allow dragging across the water we add another handler.

```javascript
canvas.onmousemove = function(e) {

  if(!e.buttons) return;

  var ray = tracer.getRayForPixel(e.x, e.y);

  var t = -tracer.eye.y / ray.y;

  var hit = tracer.eye.add(ray.multiply(t));

  water.addDrop(hit.x, hit.z, 0.02, 0.02);

};
```

Now dragging the mouse across the pool generates continuous ripples.

---

# Step 3 — Sphere–Water Interaction

The sphere should also disturb the water surface when it moves.

This requires detecting where the sphere intersects the water height field.

The water height is stored in the simulation texture.

---

# Sampling Water Height

In JavaScript we approximate the water height at the sphere position.

```
waterHeight = sample height field
```

However the demo avoids CPU readbacks.

Instead it generates disturbances directly.

---

# Sphere Displacement Force

Whenever the sphere moves we inject drops around its contact area.

```
if sphere moves

  addDrop(center.x, center.z, radius, strength)
```

This produces the ripples seen when the sphere disturbs the water.

---

# Step 4 — Updating Sphere Position

Inside the animation loop we animate the sphere.

Example motion:

```javascript
center.x = Math.sin(time)*0.5;
center.z = Math.cos(time)*0.5;
```

This causes the sphere to move around the pool.

Each frame it generates ripples.

---

# Sphere Interaction Diagram

```
     sphere
      │
      ▼
  water surface
      │
   disturbance
      │
      ▼
   ripple waves
```

The sphere effectively behaves like a moving object pushing through water.

---

# Step 5 — Applying the Sphere Force

Inside the animation loop:

```javascript
water.addDrop(
  center.x,
  center.z,
  radius * 0.5,
  0.02
);
```

This continuously injects energy into the water simulation.

---

# Full Frame Pipeline

At this point the simulation loop is complete.

```
user input
      │
      ▼
add drops
      │
      ▼
water simulation
      │
      ▼
normal update
      │
      ▼
caustics pass
      │
      ▼
render pool
render water
render sphere
```

---

# Result of Part 6

The demo is now fully interactive.

You should see:

• ripples where the mouse clicks
• waves generated as the sphere moves
• reflections and refractions reacting to the waves
• caustics moving with the water

All scene elements now respond to the same simulation state.

---

# Final Architecture

```
water simulation
(height + velocity + normals)
        │
        ▼
shared water texture
        │
  ┌─────┼────────┐
  ▼     ▼        ▼
water pool    sphere
shader shader  shader

        │
        ▼
caustics pass
        │
        ▼
final rendering
```

This architecture is what makes the demo both efficient and visually rich.

The GPU handles almost all heavy computation through fullscreen passes.

---

# Where to Go Next

Possible improvements include:

• higher resolution water simulation
• more physically accurate caustics
• refraction of scene geometry through the water volume
• porting the project to **WebGL2 or WebGPU**

These upgrades build directly on the architecture developed throughout this tutorial.

