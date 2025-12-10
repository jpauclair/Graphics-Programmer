# 30.1 Linear Algebra for Graphics

[← Back to Chapter 30](../chapters/30-Mathematics-For-Graphics.md) | [Main Index](../README.md)

Linear algebra = foundation of computer graphics (vectors, matrices, transformations). Every vertex transformation = matrix multiplication, every rotation = matrix operation, every camera projection = matrix calculation. Understanding: vector operations (dot/cross products), matrix multiplication (4x4 transforms), homogeneous coordinates (3D position + w component = enables perspective), and decomposition (extracting rotation/scale/translation from matrix).

---

## Overview

Graphics Pipeline Math: vertices start in model space (local coordinates), transform to world space (world position = model matrix × local position), then view space (camera-relative = view matrix × world position), finally clip space (projection matrix × view position = screen coordinates). Each transformation = 4x4 matrix multiplication. GPU performs billions of these per frame (1M vertices × 3 transforms = 3M matrix operations).

Matrix Composition: combine transforms (TRS = Translation × Rotation × Scale), order matters (rotate-then-translate ≠ translate-then-rotate). Optimization: pre-compute model-view-projection (MVP = projection × view × model), single matrix multiplication per vertex vs three separate.

## Key Concepts

- **Vectors**: position (x,y,z), direction (normalized = length 1), or displacement. Operations: add (v1 + v2 = component-wise), scalar multiply (c × v = scale), dot product (v1·v2 = |v1||v2|cosθ = projection), cross product (v1×v2 = perpendicular vector = normal calculation). Length: |v| = sqrt(x²+y²+z²). Normalize: v/|v| = unit vector.
- **Matrices**: 4x4 matrix = 16 floats (64 bytes). Row-major (Unity) vs column-major (OpenGL) storage affects multiplication order. Identity matrix = 1 on diagonal, 0 elsewhere (I × M = M). Inverse matrix (M⁻¹ × M = I = reverses transformation). Transpose (M^T = rows↔columns = rotations transpose = inverse for orthogonal matrices).
- **Transformations**: Translation (move object = T matrix), Rotation (rotate around axis = R matrix, Euler angles or quaternions), Scale (resize = S matrix). Combine: TRS = translate after rotate after scale (right-to-left multiplication). Homogeneous coordinates: (x,y,z,w) where w=1 for positions, w=0 for directions (directions don't translate).
- **Dot Product**: v1·v2 = x1×x2 + y1×y2 + z1×z2 = |v1||v2|cosθ. Uses: angle between vectors (cosθ = v1·v2 / |v1||v2|), projection (project v onto u = (v·u)u), backface culling (if normal·viewDir < 0 = facing away = cull). Properties: commutative (v1·v2 = v2·v1), distributive.
- **Cross Product**: v1×v2 = perpendicular vector (right-hand rule). Magnitude = |v1||v2|sinθ = parallelogram area. Uses: surface normal (triangle vertices ABC = normal = (B-A)×(C-A)), tangent/bitangent calculation (texture space basis). Properties: anti-commutative (v1×v2 = -(v2×v1)), not associative.
- **Matrix Multiplication**: C = A × B (result C[i,j] = sum of A[i,k] × B[k,j]). Not commutative (A×B ≠ B×A = order matters). Transform vertex: v' = M × v (4x4 matrix × 4x1 vector = 4x1 result). Optimization: SIMD (4-wide vector multiply = single instruction = 4x speedup).

## Best Practices

**Transform Optimization**:
- Pre-compute MVP matrix (projection × view × model = single matrix), apply once per vertex (not three separate multiplications = 3x faster).
- Cache inverse matrices (expensive to compute = O(n³), store result = reuse).
- Use quaternions for rotations (interpolation smooth, no gimbal lock, 4 floats vs 9 for 3×3 matrix).

**Numerical Stability**:
- Normalize vectors after operations (accumulated errors = length drift, renormalize periodically).
- Orthonormalize matrices (Gram-Schmidt = ensure axes perpendicular and unit length).
- Avoid near-zero denominators (divide by |v|, check |v| > epsilon first = prevent NaN).

**Platform-Specific**:
- **CPU**: use SIMD (SSE/AVX = 4-8 floats parallel, matrix multiply = 16 ops → 4-8 SIMD instructions).
- **GPU**: matrix multiplication built-in (vertex shader = automatic), leverage hardware (fast multiply-add = matrix ops nearly free).
- **Mobile**: reduced precision (16-bit float = half precision = 2x faster, acceptable for most graphics).

## Common Pitfalls

**Wrong Multiplication Order**: developer multiplies T × R × S (expects rotate then translate), but gets translate-rotate-scale (wrong order). Symptom: object rotates around wrong pivot (world origin instead of center). Solution: right-to-left multiplication (TRS = apply S first, then R, then T).

**Non-Normalized Vectors**: dot product computed without normalizing (v1·v2 used as cosθ, but |v1|≠1 or |v2|≠1 = wrong angle). Symptom: lighting incorrect (diffuse = dot(N,L) but N or L not normalized = wrong intensity). Solution: normalize before dot product (v1/|v1| · v2/|v2|).

**Gimbal Lock**: Euler angles (pitch,yaw,roll) lose degree of freedom (two axes align = can't rotate independently). Symptom: camera rotation jerky (rotating pitch = yaw changes unexpectedly). Solution: use quaternions (no gimbal lock, smooth interpolation).

## Tools & Workflow

**Vector Operations** (C#):
```csharp
Vector3 v1 = new Vector3(1, 0, 0);
Vector3 v2 = new Vector3(0, 1, 0);
float dot = Vector3.Dot(v1, v2); // = 0 (perpendicular)
Vector3 cross = Vector3.Cross(v1, v2); // = (0,0,1) (perpendicular to both)
Vector3 normalized = v1.normalized; // = v1/|v1|
```

**Matrix Multiplication**:
```csharp
Matrix4x4 model = Matrix4x4.TRS(position, rotation, scale);
Matrix4x4 view = camera.worldToCameraMatrix;
Matrix4x4 projection = camera.projectionMatrix;
Matrix4x4 mvp = projection * view * model; // Pre-compute
Vector4 clipPos = mvp * new Vector4(vertex.x, vertex.y, vertex.z, 1.0f);
```

**Quaternion Rotation**:
```csharp
Quaternion rot = Quaternion.Euler(45, 0, 0); // 45° pitch
Vector3 rotated = rot * Vector3.forward; // Rotate forward vector
Quaternion slerp = Quaternion.Slerp(rot1, rot2, t); // Smooth interpolation
```

## Related Topics

- [30.2 Color Spaces](30-02-Color-Spaces.md) - Color mathematics
- [30.3 Coordinate Systems](30-03-Coordinate-Systems.md) - Space transformations
- [01.2 Vertex Processing](01-02-Vertex-Processing.md) - GPU vertex transforms
- [16.1 Camera Fundamentals](16-01-Camera-Fundamentals.md) - Camera matrices

---

[Previous: Chapter 29](../chapters/29-Case-Studies-And-Best-Practices.md) | [Next: 30.2 Color Spaces →](30-02-Color-Spaces.md)
