# 30.3 Coordinate Systems

[← Back to Chapter 30](../chapters/30-Mathematics-For-Graphics.md) | [Main Index](../README.md)

Coordinate systems = reference frames for vertex positions (model space = object local, world space = scene global, view space = camera relative, clip space = projection applied). Every vertex transformed through pipeline: model → world → view → clip → NDC → screen. Each transformation = matrix multiplication (MVP matrix = projection × view × model), each space has purpose (model = artist authoring, world = physics/gameplay, view = lighting, clip = GPU rasterization).

---

## Overview

Transformation Pipeline: vertex starts in model space (local coordinates, origin at object pivot), transform by model matrix → world space (scene position), transform by view matrix → view space (camera-relative, camera at origin looking down -Z), transform by projection matrix → clip space (homogeneous coordinates, frustum to cube), perspective divide → NDC (Normalized Device Coordinates, [-1,1] cube), viewport transform → screen space (pixel coordinates).

Handedness: Unity = left-handed (positive Z forward), OpenGL/Vulkan = right-handed (negative Z forward). Affects cross product (normal calculation = flipped), projection matrix (Z range different), and winding order (front-facing = clockwise vs counter-clockwise). DirectX = left-handed, OpenGL/Metal/Vulkan = right-handed, Unity abstracts differences.

## Key Concepts

- **Model Space (Local Space)**: object's local coordinates, origin at pivot. Mesh vertices authored in model space (artist exports from DCC). Transform to world: vertex_world = model_matrix × vertex_local (position, rotate, scale). Used by: mesh authoring, procedural generation, instancing (same mesh different transforms).
- **World Space**: scene global coordinates, origin at (0,0,0). All objects positioned here (gameplay, physics, lighting). Transform to world: model matrix (TRS = translate × rotate × scale, applied right-to-left). Used by: collision detection, AI, lighting calculations (light positions in world space).
- **View Space (Camera Space)**: camera-relative coordinates, camera at origin looking down -Z (Unity) or +Z (OpenGL). Transform: view matrix = inverse camera transform (camera world → view = move world opposite camera). Z-axis = depth (near objects negative Z, far objects larger). Used by: depth sorting, view-dependent effects (fog, reflections).
- **Clip Space**: after projection matrix, homogeneous coordinates (x,y,z,w) where w ≠ 1. Frustum mapped to clip cube (x,y,z ∈ [-w,w]). Vertices outside clipped (discarded if any component > w). Transform: clip = projection × view × model × vertex (MVP matrix). GPU performs: perspective divide (x/w, y/w, z/w) → NDC.
- **NDC (Normalized Device Coordinates)**: after perspective divide, normalized cube. Unity/DirectX: x,y ∈ [-1,1], z ∈ [0,1]. OpenGL: x,y,z ∈ [-1,1]. Used by: rasterizer (converts to screen pixels), depth buffer (z stored, z-test performed).
- **Screen Space (Viewport Space)**: pixel coordinates, origin top-left (0,0) or bottom-left (OpenGL). Viewport transform: x_screen = (x_ndc + 1) × width/2, y_screen = (y_ndc + 1) × height/2. Z → depth buffer [0,1]. Used by: rasterization (pixel shading), post-processing (screen-space effects).

## Best Practices

**Matrix Composition**:
- Precompute MVP: `Matrix4x4 mvp = projection * view * model;` (CPU-side, pass to shader).
- Shader multiplication: `float4 clipPos = mul(UNITY_MATRIX_MVP, float4(vertexPos, 1.0));` (Unity built-in).
- Order matters: TRS = translate × rotate × scale (right-to-left), rotate first, then translate.

**Coordinate Conversion**:
- World → View: `Vector3 viewPos = Camera.main.worldToCameraMatrix.MultiplyPoint(worldPos);`
- View → Clip: `Vector4 clipPos = Camera.main.projectionMatrix * new Vector4(viewPos.x, viewPos.y, viewPos.z, 1);`
- Clip → NDC: `Vector3 ndc = new Vector3(clipPos.x/clipPos.w, clipPos.y/clipPos.w, clipPos.z/clipPos.w);`

**Depth Precision**:
- Reverse-Z: map far plane to 0, near to 1 (better float precision at distance).
- Unity default: z ∈ [0,1], reverse-Z optional (improve far plane precision).
- Avoid: extremely large far plane (precision loss = Z-fighting).

**Platform-Specific**:
- **PC/Console**: full precision depth (24-bit or 32-bit float).
- **Mobile**: 16-bit depth common (reduce far plane = improve precision).
- **VR**: stereo rendering (separate view matrices per eye, world space shared).

## Common Pitfalls

**Wrong Multiplication Order**: developer multiplies `model * view * projection` (incorrect, backwards). Symptom: vertices wrong position (object appears warped). Solution: correct order = `projection * view * model` (right-to-left composition).

**Forgetting w=1 for Positions**: shader multiplies vector3 (w=0 assumed = direction, not position). Symptom: vertices at origin (translation ignored). Solution: use `float4(vertexPos, 1.0)` for positions, `float4(normalDir, 0.0)` for directions.

**Mixing Coordinate Spaces**: lighting calculation uses world-space normals with view-space light direction. Symptom: incorrect lighting (wrong dot product). Solution: ensure all vectors in same space (typically world space for lighting, or transform light to view space).

## Tools & Workflow

**Unity Coordinate Conversion** (C#):
```csharp
// World → View
Vector3 viewPos = Camera.main.worldToCameraMatrix.MultiplyPoint(worldPos);

// World → Screen
Vector3 screenPos = Camera.main.WorldToScreenPoint(worldPos);

// Screen → World (ray casting)
Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
```

**Shader Coordinate Transformation** (HLSL):
```hlsl
// Vertex shader
float4 clipPos = UnityObjectToClipPos(v.vertex); // Model → Clip (Unity built-in)

// Manual MVP
float4 worldPos = mul(unity_ObjectToWorld, v.vertex); // Model → World
float4 viewPos = mul(UNITY_MATRIX_V, worldPos); // World → View
float4 clipPos = mul(UNITY_MATRIX_P, viewPos); // View → Clip
```

**NDC to Screen** (Fragment shader):
```hlsl
// Fragment shader, clipPos passed from vertex
float2 screenUV = (i.clipPos.xy / i.clipPos.w) * 0.5 + 0.5; // NDC → [0,1]
float depth = i.clipPos.z / i.clipPos.w; // Normalized depth
```

**Custom View Matrix** (C#):
```csharp
Matrix4x4 view = Matrix4x4.TRS(cameraPos, cameraRot, Vector3.one).inverse;
// Or use LookAt
Matrix4x4 view = Matrix4x4.LookAt(cameraPos, targetPos, Vector3.up);
```

## Related Topics

- [30.1 Linear Algebra](30-01-Linear-Algebra.md) - Matrix transformations
- [01.2 Vertex Processing](01-02-Vertex-Processing.md) - Vertex transformation pipeline
- [16.1 Camera Fundamentals](16-01-Camera-Fundamentals.md) - View and projection matrices
- [13.2 Shadow Mapping](13-02-Shadow-Mapping.md) - Light space transformations

---

[← Previous: 30.2 Color Spaces](30-02-Color-Spaces.md) | [Next: 30.4 Geometry Mathematics →](30-04-Geometry-Mathematics.md)
