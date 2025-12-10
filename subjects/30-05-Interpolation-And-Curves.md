# 30.5 Interpolation and Curves

[← Back to Chapter 30](../chapters/30-Mathematics-For-Graphics.md) | [Main Index](../README.md)

Interpolation = calculate intermediate values between known points (lerp = linear, smoothstep = smooth acceleration/deceleration, ease functions = animation timing). Curves = paths defined by control points (Bezier = 2D/3D curves, splines = smooth connected segments). Used by: animation (keyframe blending), camera paths (smooth movement), procedural generation (terrain curves), UI (smooth transitions).

---

## Overview

Linear Interpolation (Lerp): simplest interpolation, straight line between points. Formula: lerp(a, b, t) = (1-t)×a + t×b where t ∈ [0,1]. t=0 → a, t=0.5 → midpoint, t=1 → b. Constant velocity (no acceleration). Used by: position blending, color gradients, parameter animation.

Smooth Interpolation: acceleration/deceleration at endpoints (ease-in/ease-out). Smoothstep: t² × (3 - 2t) (S-curve, smooth at 0 and 1). Smootherstep: t³ × (t × (6t - 15) + 10) (smoother derivatives). Ease functions: cubic, quartic, exponential (different acceleration profiles). Used by: UI animations, camera transitions, gameplay feel.

## Key Concepts

- **Linear Interpolation (Lerp)**: lerp(a, b, t) = a + (b - a) × t = (1-t)×a + t×b. Constant velocity, no acceleration. Multi-dimensional: lerp vectors/colors component-wise. Clamped: Mathf.Lerp (t clamped [0,1]), unclamped: LerpUnclamped (extrapolate outside range). Used by: Vector3.Lerp, Color.Lerp, Quaternion.Lerp (rotation).
- **Smoothstep**: smoothstep(a, b, t) = lerp(a, b, smooth(t)) where smooth(t) = t² × (3 - 2t) (Hermite interpolation). Derivative zero at endpoints (smooth acceleration/deceleration). Unity: Mathf.SmoothStep(a, b, t). Alternative: smootherstep = t³ × (t × (6t - 15) + 10) (C2 continuous = smoother). Used by: camera easing, UI fades, animation blending.
- **Bezier Curves**: parametric curves, control points define shape. Quadratic Bezier (3 points): B(t) = (1-t)²×P0 + 2(1-t)t×P1 + t²×P2. Cubic Bezier (4 points): B(t) = (1-t)³×P0 + 3(1-t)²t×P1 + 3(1-t)t²×P2 + t³×P3. P0, P3 = endpoints, P1, P2 = control points (shape curve). Used by: vector graphics, animation curves (Unity AnimationCurve), font rendering.
- **Catmull-Rom Splines**: smooth curve through all control points (interpolating spline). Formula: P(t) = 0.5 × [(2P1) + (-P0+P2)t + (2P0-5P1+4P2-P3)t² + (-P0+3P1-3P2+P3)t³]. Requires 4 points (P0, P1, P2, P3) to calculate segment P1 → P2. Tangent at P1 = (P2-P0)/2 (automatic tangents). Used by: camera paths, smooth trajectories, procedural splines.
- **Slerp (Spherical Linear Interpolation)**: interpolate on sphere surface (rotation, direction). Quaternion.Slerp(q1, q2, t) = sin((1-t)θ)/sinθ × q1 + sin(tθ)/sinθ × q2 where θ = angle between quaternions. Constant angular velocity (unlike Quaternion.Lerp = linear blend, then normalize). Used by: rotation interpolation, camera look-at, directional blending.
- **Ease Functions**: timing curves for animation feel. Ease-in: slow start, fast end (t²). Ease-out: fast start, slow end (1 - (1-t)²). Ease-in-out: smooth both ends (smoothstep). Exponential: dramatic acceleration (2^(10×(t-1))). Bounce: simulate bounce physics. Used by: UI tweening (DOTween, LeanTween), gameplay animation, screen transitions.

## Best Practices

**Interpolation Selection**:
- Constant velocity: Lerp (simple, predictable).
- Smooth motion: SmoothStep (ease-in/out, natural feel).
- Rotation: Slerp (correct angular interpolation, avoid gimbal lock).
- Animation curves: Bezier or Unity AnimationCurve (artist-defined timing).

**Performance Optimization**:
- Lerp: cheapest (one multiply, one add per component).
- SmoothStep: moderate (polynomial evaluation, ~5 ops).
- Slerp: expensive (sin/cos, normalize), use Lerp for small angles (<10°).
- AnimationCurve: CPU evaluation, cache results or use shader curve texture.

**Numerical Stability**:
- Clamp t to [0,1] unless extrapolation intended (avoid overshooting).
- Normalize quaternions after Slerp (accumulation errors = drift).
- Avoid division by zero: check sin(θ) ≈ 0 in Slerp (fallback to Lerp).

**Platform-Specific**:
- **PC/Console**: all interpolation fast, use Slerp freely.
- **Mobile**: prefer Lerp over Slerp (avoid trigonometry), use SmoothStep sparingly.
- **GPU**: SmoothStep hardware instruction (free in shader), Bezier evaluation in vertex shader.

## Common Pitfalls

**Lerp in Update**: developer calls `transform.position = Vector3.Lerp(start, end, t);` with fixed t (e.g., 0.5). Symptom: object moves halfway each frame (exponential decay, never reaches target). Solution: use Time.deltaTime: `t += Time.deltaTime * speed;` (linear progression), or use `Vector3.MoveTowards`.

**Quaternion Lerp for Large Angles**: using Quaternion.Lerp instead of Slerp for 180° rotation. Symptom: rotation takes shortest path through quaternion space (not spherical = weird interpolation). Solution: Quaternion.Slerp for angles >10°, or normalize after Lerp.

**AnimationCurve Performance**: evaluating AnimationCurve.Evaluate every frame for many objects. Symptom: CPU spike (curve evaluation = binary search keyframes). Solution: bake curve to texture (sample in shader), or cache evaluated values (lookup table).

## Tools & Workflow

**Basic Interpolation** (C#):
```csharp
// Linear interpolation
float value = Mathf.Lerp(start, end, t);
Vector3 pos = Vector3.Lerp(startPos, endPos, t);
Color color = Color.Lerp(color1, color2, t);

// Smooth interpolation
float smooth = Mathf.SmoothStep(start, end, t);

// Rotation interpolation
Quaternion rot = Quaternion.Slerp(startRot, endRot, t);
```

**Cubic Bezier** (C#):
```csharp
Vector3 CubicBezier(Vector3 p0, Vector3 p1, Vector3 p2, Vector3 p3, float t)
{
    float u = 1 - t;
    float tt = t * t;
    float uu = u * u;
    float uuu = uu * u;
    float ttt = tt * t;
    
    Vector3 p = uuu * p0; // (1-t)³ × P0
    p += 3 * uu * t * p1; // 3(1-t)²t × P1
    p += 3 * u * tt * p2; // 3(1-t)t² × P2
    p += ttt * p3;        // t³ × P3
    
    return p;
}
```

**Catmull-Rom Spline** (C#):
```csharp
Vector3 CatmullRom(Vector3 p0, Vector3 p1, Vector3 p2, Vector3 p3, float t)
{
    float t2 = t * t;
    float t3 = t2 * t;
    
    return 0.5f * (
        (2.0f * p1) +
        (-p0 + p2) * t +
        (2.0f * p0 - 5.0f * p1 + 4.0f * p2 - p3) * t2 +
        (-p0 + 3.0f * p1 - 3.0f * p2 + p3) * t3
    );
}
```

**Unity AnimationCurve**:
```csharp
[SerializeField] AnimationCurve curve = AnimationCurve.EaseInOut(0, 0, 1, 1);

void Update()
{
    float t = Time.time % 1.0f;
    float value = curve.Evaluate(t); // Evaluate curve at t
    transform.position = Vector3.Lerp(start, end, value);
}
```

**Shader SmoothStep** (HLSL):
```hlsl
// Hardware smoothstep (free instruction)
float t = smoothstep(0.0, 1.0, input);

// Manual implementation
float smooth(float t) {
    return t * t * (3.0 - 2.0 * t);
}
```

## Related Topics

- [30.1 Linear Algebra](30-01-Linear-Algebra.md) - Vector interpolation
- [16.3 Camera Animation](16-03-Camera-Animation.md) - Camera path interpolation
- [18.3 Animation Blending](18-03-Animation-Blending.md) - Keyframe interpolation
- [30.2 Color Spaces](30-02-Color-Spaces.md) - Color interpolation

---

[← Previous: 30.4 Geometry Mathematics](30-04-Geometry-Mathematics.md) | [Next: Chapter 31 - Third-Party Tools →](../chapters/31-Third-Party-Tools.md)
