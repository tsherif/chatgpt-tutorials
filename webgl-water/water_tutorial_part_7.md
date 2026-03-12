# WebGL Water Tutorial — Part 7: Deep Dive Into the Simulation

This final part steps back from the implementation and explains **why the system works**.

The earlier parts focused on building the demo step‑by‑step. Here we examine the underlying math and GPU techniques that make the approach efficient.

Topics covered:

• the wave equation approximation used by the simulation
• why the shader stores **height and velocity**
• why **ping‑pong textures** are required
• how the **normal reconstruction** works
• performance characteristics of the approach

---

# The Physical Model

The water simulation approximates a **2‑D wave equation**.

The continuous equation for wave motion is

```
∂²h/∂t² = c² ∇²h
```

Where

```
h(x,z,t) = surface height
c       = wave speed
∇²h     = spatial curvature (Laplacian)
```

Interpretation:

• if a point is higher than its neighbors, it accelerates downward
• if a point is lower than neighbors, it accelerates upward

This causes oscillating waves.

---

# Discrete Grid Representation

The simulation runs on a texture grid.

```
water texture

+---+---+---+
|   |   |   |
+---+---+---+
|   | X |   |
+---+---+---+
|   |   |   |
+---+---+---+
```

Each texel represents a small patch of water.

The Laplacian is approximated using the four neighbors.

```
average = (left + right + up + down) / 4
```

This produces the curvature estimate used by the simulation.

---

# Height and Velocity

Instead of directly integrating the wave equation, the demo stores two values per texel:

```
R = height
G = vertical velocity
```

The update shader performs two steps:

1) update velocity from curvature

```
velocity += (averageHeight - height) * k
```

2) update height from velocity

```
height += velocity
```

This is a simple form of **semi‑implicit Euler integration**.

---

# Damping

Without damping, waves would continue forever.

The shader applies a small decay.

```
velocity *= 0.995
```

This slowly removes energy from the system.

The value is tuned to look natural.

---

# Simulation Update Shader

The key part of the simulation shader from Part 1:

```glsl
float average = (
 texture2D(texture,coord-dx).r +
 texture2D(texture,coord+dx).r +
 texture2D(texture,coord-dy).r +
 texture2D(texture,coord+dy).r
)*0.25;

info.g += (average - info.r) * 2.0;
info.g *= 0.995;
info.r += info.g;
```

This implements the discrete wave model described above.

---

# Why Ping‑Pong Textures Are Required

A GPU fragment shader cannot safely read and write the same texture simultaneously.

Instead we alternate between two textures.

```
textureA  (current state)
     │
     ▼
update shader
     │
     ▼
textureB  (next state)

swap(A,B)
```

This technique is called **ping‑pong rendering**.

It is widely used for GPU simulations.

---

# Why the Simulation Runs on the GPU

Advantages of GPU simulation:

• massively parallel
• every texel updates simultaneously
• no CPU memory transfers

The simulation cost scales with **texture size**, not scene complexity.

---

# Normal Reconstruction

The water texture stores only two components of the surface normal.

```
B = normal.x
A = normal.z
```

The missing Y component is reconstructed.

```
y = sqrt(1 − x² − z²)
```

This works because normals are unit vectors.

```
x² + y² + z² = 1
```

Storing two components instead of three saves texture bandwidth.

---

# Why Normals Are Stored in the Simulation Texture

Normals must be recomputed every frame because the water surface changes.

Instead of recomputing them in every rendering shader, the demo computes them once in a dedicated pass.

```
water height
     │
     ▼
normal pass
     │
     ▼
water texture now stores normals
```

All renderers then read the same data.

---

# Performance Characteristics

The simulation is extremely efficient.

Typical cost per frame:

```
2 simulation passes
1 normal pass
1 caustics pass
```

All are fullscreen quad renders.

Even on modest hardware this runs easily in real time.

---

# Why the Demo Looks So Good

Several small tricks combine to produce convincing water:

• reflection + refraction
• Fresnel blending
• caustics
• dynamic ripples
• environment lighting

None of these effects are individually expensive, but together they produce strong realism.

---

# Possible Extensions

Many improvements can be built on the same architecture.

Examples:

### Higher Resolution Simulation

Increasing the water texture size produces finer waves.

### WebGL2 / WebGPU Port

Modern APIs allow:

• compute shaders
• better floating‑point support
• more efficient pipelines

### Multiple Light Sources

The caustics pass could accumulate contributions from several lights.

### Better Wave Physics

More accurate integration methods or additional wave terms could improve realism.

---

# Final Summary

The demo demonstrates an elegant pattern for real‑time graphics:

```
GPU simulation
        │
        ▼
shared state texture
        │
        ▼
multiple rendering passes
```

Because every stage reads the same simulation data, the entire scene remains visually consistent.

The result is a rich, dynamic water simulation built from a relatively small amount of code.

---

End of Tutorial

