# 30.4 Geometry Mathematics

[← Back to Chapter 30](../chapters/30-Mathematics-For-Graphics.md) | [Main Index](../README.md)

Geometry mathematics = spatial calculations (triangle area, point-in-triangle test, ray-triangle intersection). Graphics pipeline built on triangles (every mesh = triangles, rasterization = fill triangles). Math operations: barycentric coordinates (interpolate vertex attributes), distance calculations (culling, LOD switching), intersection tests (picking, collision), plane equations (clipping, reflections).

---

## Overview

Triangle Mathematics: triangles defined by 3 vertices (v0, v1, v2), counter-clockwise = front-facing (Unity default). Area calculation = 0.5 × |cross(v1-v0, v2-v0)| (cross product magnitude). Normal = cross(v1-v0, v2-v0) normalized (perpendicular to triangle). Barycentric coordinates (u,v,w) where u+v+w=1 = interpolate vertex data (point P = u×v0 + v×v1 + w×v2).

Intersection Tests: ray-triangle (ray casting, picking), ray-sphere (bounding volume test), AABB-AABB (collision detection), plane-point (distance to plane). Used by: picking (mouse click → world position), culling (frustum test = is object visible), collision (physics, raycasting).

## Key Concepts

- **Barycentric Coordinates**: represent point in triangle as weighted sum (u,v,w) where u+v+w=1. Point P = u×v0 + v×v1 + w×v2. If u,v,w ∈ [0,1] → inside triangle. Calculation: solve linear system (cross products, area ratios). Used by: texture coordinate interpolation (GPU rasterizer), attribute blending (normals, colors), ray-triangle intersection.
- **Triangle Area**: Area = 0.5 × |cross(v1-v0, v2-v0)| = half the parallelogram area. Alternative: Heron's formula = sqrt(s(s-a)(s-b)(s-c)) where s=(a+b+c)/2 (slower, avoid). Signed area = 0.5 × cross (positive = CCW, negative = CW). Used by: winding order test, barycentric calculation, mesh validation.
- **Ray-Triangle Intersection**: ray origin O, direction D, triangle (v0,v1,v2). Möller-Trumbore algorithm: solve P = O + t×D = u×v0 + v×v1 + w×v2 (3 equations, 3 unknowns). Result: t (hit distance), u,v (barycentric). Early-out: determinant near zero = parallel. Used by: ray casting, mouse picking, physics raycasting.
- **Distance Calculations**: Point-to-plane = dot(P - planeOrigin, planeNormal) (signed distance). Point-to-line = cross product perpendicular distance. Point-to-triangle = barycentric clamp + distance to plane. Used by: culling (frustum distance), LOD (distance-based switching), fog (depth-based).
- **Plane Equation**: ax + by + cz + d = 0 where (a,b,c) = normal, d = -dot(normal, pointOnPlane). Point substitution: dot(normal, P) + d (>0 = front, <0 = back, =0 = on plane). Used by: clipping (frustum planes), reflections (mirror plane), portals.
- **AABB (Axis-Aligned Bounding Box)**: box defined by min/max corners (minX, minY, minZ) and (maxX, maxY, maxZ). Intersection test: overlap all axes (separating axis theorem). AABB-AABB: if (max1.x < min2.x || min1.x > max2.x) → no overlap (repeat for Y, Z). Used by: broad-phase collision, frustum culling, spatial partitioning (octree, BVH).

## Best Practices

**Intersection Optimization**:
- Ray-triangle: precompute edge vectors (v1-v0, v2-v0), reuse for multiple rays.
- AABB tests: early-out on first axis failure (fastest reject).
- Sphere bounds: fast distance check (|center - point| < radius) before expensive tests.

**Numerical Stability**:
- Avoid zero-area triangles (degenerate = collapsed vertices, cross product → 0).
- Epsilon comparisons: `if (Mathf.Abs(distance) < 0.0001f)` (avoid exact zero due to float precision).
- Normalize vectors: dot product requires unit vectors (incorrect magnitude = wrong result).

**Barycentric Interpolation**:
- GPU automatic: fragment shader receives interpolated attributes (no manual calculation).
- CPU ray-tracing: calculate barycentric (u,v), interpolate manually: `value = u×v0 + v×v1 + (1-u-v)×v2`.

**Platform-Specific**:
- **PC/Console**: full precision (32-bit float), fast cross products (SIMD).
- **Mobile**: half-precision sufficient for barycentric (16-bit), avoid sqrt (expensive on mobile GPU).
- **GPU**: hardware interpolation (free barycentric for fragment attributes).

## Common Pitfalls

**Wrong Winding Order**: triangle defined (v0,v2,v1) instead of (v0,v1,v2) (clockwise instead of counter-clockwise). Symptom: back-face culled incorrectly (triangle invisible from front). Solution: ensure CCW vertex order, check mesh importer settings.

**Non-Normalized Normals**: developer uses cross product without normalizing (|cross(v1-v0, v2-v0)| > 1). Symptom: incorrect lighting (dot product wrong, specular too bright/dark). Solution: normalize all normals: `normal = Vector3.Cross(e1, e2).normalized;`

**Barycentric Outside [0,1]**: point P outside triangle (u or v < 0, or u+v > 1). Symptom: incorrect interpolation (wrong texture coordinates, negative weights). Solution: clamp to triangle (closest point on triangle), or reject (point not on surface).

## Tools & Workflow

**Barycentric Coordinates** (C#):
```csharp
// Calculate barycentric coordinates of point P in triangle (v0, v1, v2)
Vector3 v0v1 = v1 - v0;
Vector3 v0v2 = v2 - v0;
Vector3 v0p = P - v0;

float d00 = Vector3.Dot(v0v1, v0v1);
float d01 = Vector3.Dot(v0v1, v0v2);
float d11 = Vector3.Dot(v0v2, v0v2);
float d20 = Vector3.Dot(v0p, v0v1);
float d21 = Vector3.Dot(v0p, v0v2);

float denom = d00 * d11 - d01 * d01;
float v = (d11 * d20 - d01 * d21) / denom;
float w = (d00 * d21 - d01 * d20) / denom;
float u = 1.0f - v - w;

// Check if inside triangle
bool inside = (u >= 0 && v >= 0 && w >= 0);
```

**Ray-Triangle Intersection (Möller-Trumbore)**:
```csharp
bool RayIntersectsTriangle(Vector3 rayOrigin, Vector3 rayDir, Vector3 v0, Vector3 v1, Vector3 v2, out float t)
{
    Vector3 edge1 = v1 - v0;
    Vector3 edge2 = v2 - v0;
    Vector3 h = Vector3.Cross(rayDir, edge2);
    float a = Vector3.Dot(edge1, h);
    
    if (Mathf.Abs(a) < 0.00001f) return false; // Parallel
    
    float f = 1.0f / a;
    Vector3 s = rayOrigin - v0;
    float u = f * Vector3.Dot(s, h);
    
    if (u < 0.0f || u > 1.0f) return false;
    
    Vector3 q = Vector3.Cross(s, edge1);
    float v = f * Vector3.Dot(rayDir, q);
    
    if (v < 0.0f || u + v > 1.0f) return false;
    
    t = f * Vector3.Dot(edge2, q);
    return t > 0.00001f; // Hit
}
```

**AABB Intersection**:
```csharp
bool AABBIntersect(Bounds a, Bounds b)
{
    return (a.min.x <= b.max.x && a.max.x >= b.min.x) &&
           (a.min.y <= b.max.y && a.max.y >= b.min.y) &&
           (a.min.z <= b.max.z && a.max.z >= b.min.z);
}
```

**Plane Distance**:
```csharp
// Plane defined by normal and point
float DistanceToPlane(Vector3 point, Vector3 planeNormal, Vector3 planePoint)
{
    float d = -Vector3.Dot(planeNormal, planePoint);
    return Vector3.Dot(planeNormal, point) + d;
}
```

## Related Topics

- [30.1 Linear Algebra](30-01-Linear-Algebra.md) - Cross product, dot product
- [01.2 Vertex Processing](01-02-Vertex-Processing.md) - Vertex attribute interpolation
- [06.2 CPU Culling Techniques](06-02-CPU-Culling-Techniques.md) - Frustum culling math
- [17.2 LOD Selection](17-02-LOD-Selection.md) - Distance calculations

---

[← Previous: 30.3 Coordinate Systems](30-03-Coordinate-Systems.md) | [Next: 30.5 Interpolation And Curves →](30-05-Interpolation-And-Curves.md)
