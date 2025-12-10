# 29.4 Platform Certification Requirements

[← Back to Chapter 29](../chapters/29-Case-Studies-And-Best-Practices.md) | [Main Index](../README.md)

Console manufacturers (Sony, Microsoft, Nintendo) require performance certification before game release. Requirements vary by platform and release timing. This section details technical performance gates: frame rate targets, frame time variance limits, load time maximums, memory constraints, and platform-specific guidelines.

---

## Overview

Certification ensures minimum user experience. Sony: "no frame drops >16ms sustained" (maintains 60fps). Microsoft: similar requirements, but more flexible for specific game designs. Nintendo: Switch lower bar (30fps acceptable), but battery life matters.

Certification failure = delayed release. Plan 2-3 weeks buffer for rework. Failing certification post-ship risks delisting (game removed from store).

## Key Concepts

- **Frame Rate Gate**: Target frame rate (60fps, 30fps, 120fps) must be maintained. Allowed variance: 1 frame drop per 5 minutes for PS5. Sustained frame rate required (not just average).

- **Load Time Gate**: Initial load <2 minutes (from disc insertion to main menu). Level load <30 seconds (streaming levels). Boot time <20 seconds from power-on.

- **Memory Gate**: Game cannot exceed memory budget. PS5: 10GB user memory (rest OS). Xbox Series X: 10GB fast RAM + 6GB slow RAM (must manage correctly).

- **Crash Gate**: No crashes over 2-hour gameplay test. Stress test: 20 maps, 3 hours each = must not crash.

- **Online Stability**: Multiplayer certification: server must handle maximum concurrent players + 30% headroom. No disconnects during stress test.

## Certification Requirements by Platform

**PlayStation 5 (Sony):**

*Performance Requirements:*
- Frame rate target: 60fps (most games). Or 30fps if visual quality justifies. Or 120fps (esports). Target consistency: p99 frame time within 5% of target. Example: 60fps target = 16.67ms, allow up to 17.5ms p99.
- Resolution: 1440p minimum (1080p only if special artistic reason). 4K acceptable if frame rate maintained.
- No sustained frame drops: 1 frame drop per 5 minutes max. Two consecutive frame drops = failure.
- Load time: Initial load <2 minutes. Level load <30 seconds (including asset streaming).

*Testing Protocol:*
- Gameplay test: 20 maps, 30 minutes each = 10 hours. Frame rate profiling entire time. Any sustained drop = fail.
- Stress test: max difficulty, max active elements (enemies, particles). Must maintain frame rate.
- Memory test: play until memory peak recorded. If exceeds budget, fail.

*Failure Examples:*
- Game maintains 60fps for 10 minutes, then drops to 30fps when loading next level. Fail (must stream seamlessly).
- Game crashes once per 5 hours gameplay. Fail (crash gate: zero crashes).

*Workaround Examples:*
- If 60fps unachievable with full features: reduce to 30fps (with high visual quality disclosure). Certification allows if consistently 30fps.
- If load time >30s unavoidable: add gameplay during loading (splash screens, loading screens with art). Not a workaround, but UX mitigation.

**Xbox Series X (Microsoft):**

*Performance Requirements:*
- Frame rate target: similar to PS5 (60fps or 30fps, consistency required).
- Variable refresh rate (VRR): Xbox supports VRR (monitor frame rate to match GPU output, e.g., 58fps if GPU can't hit 60fps). If game uses VRR, frame rate variance allowed (up to ±2fps around target). Without VRR: strict frame lock.
- Backward compatibility: Xbox One version must also pass certification (easier standard: 1080p 30fps OK).
- Quick Resume: game must save state for instant resume (support required for certification). Impact: memory footprint (state must fit in quick resume buffer).

*Testing Protocol:*
- Similar to PS5 (20 maps, 10 hours gameplay test).
- VRR testing: if game supports VRR, profiler shows allowed variance. If not, strict frame lock.

**Nintendo Switch:**

*Performance Requirements:*
- Frame rate target: 30fps typical. Some titles 60fps (indie games). 30fps p99 allowed (more lenient than PlayStation).
- Handheld vs docked: docked mode must maintain target. Handheld can reduce resolution/fidelity. Battery life: game must run 3+ hours handheld without pause.
- Load time: <2 minutes (similar to home consoles).
- Memory: 4GB available (more constrained than PS5/Xbox).

*Testing Protocol:*
- Gameplay test: 20 levels, 20 minutes each. Handheld + docked mode both tested.
- Battery test: play 3 hours handheld, measure battery drain. If battery < 15% at 3 hours, fail.

## Best Practices

**Certification Readiness Checklist:**

```markdown
- [ ] Frame rate test: play 10 hours, no sustained drops
- [ ] Load time test: all levels <30s load (measure from level select to gameplay)
- [ ] Crash test: 20 hours gameplay, zero crashes
- [ ] Memory test: peak memory usage < budget
- [ ] Online stability (if multiplayer): 100+ concurrent players, 2 hours, no disconnects
- [ ] Audio/localization: all text languages supported, no corrupted audio
- [ ] Controller support: all controller layouts supported, all buttons functional
- [ ] Accessibility: colorblind mode, text size options, subtitle support
```

**Pre-Certification Profiling:**

```csharp
public class CertificationTester
{
    private List<float> m_FrameTimes = new List<float>();
    private int m_FrameCount = 0;
    private const int TARGET_FPS = 60;
    private const float TARGET_FRAME_TIME = 1f / TARGET_FPS;  // 0.0167s
    private const float ALLOWED_VARIANCE = 0.005f;  // ±5%
    
    void Update()
    {
        float frameTime = Time.deltaTime;
        m_FrameTimes.Add(frameTime);
        m_FrameCount++;
        
        if (m_FrameCount == 36000)  // 10 hours at 60fps
            AnalyzeCertification();
    }
    
    void AnalyzeCertification()
    {
        float p99 = m_FrameTimes.OrderBy(x => x).ElementAt((int)(m_FrameTimes.Count * 0.99)).First();
        float allowed = TARGET_FRAME_TIME + ALLOWED_VARIANCE;
        
        if (p99 < allowed)
            Debug.Log($"PASS: p99 frame time {p99*1000}ms < {allowed*1000}ms");
        else
            Debug.LogError($"FAIL: p99 frame time {p99*1000}ms > {allowed*1000}ms");
        
        // Check for sustained drops (>2 consecutive frames >threshold)
        int consecutiveDrops = 0;
        for (int i = 0; i < m_FrameTimes.Count; i++)
        {
            if (m_FrameTimes[i] > allowed)
                consecutiveDrops++;
            else
                consecutiveDrops = 0;
            
            if (consecutiveDrops > 2)
                Debug.LogError($"FAIL: Sustained frame drop at frame {i}");
        }
    }
}
```

## Platform-Specific Guidance

**PlayStation 5:**
- SSD I/O: Leverage 5.5GB/s for aggressive streaming. Certification expects instant load transitions.
- Ray tracing: Encouraged for visual showcase. Common: 1 bounce reflection rays + ray-traced shadows (sp at 60fps with denoising).
- DualSense features: haptic feedback expected (certification bonus, not required). But if used, must be functional.

**Xbox Series X:**
- Velocity Architecture (compression): Use for faster decompression. Custom compression APIs available.
- Smart Delivery: game must support both Xbox Series X and Xbox One versions (one purchase). Certification gates both versions.
- Game Pass: if on Game Pass, telemetry expected (Microsoft collects aggregated play data).

**Nintendo Switch:**
- Docked/handheld: Test both modes. Can reduce graphics handheld, but art style must remain consistent.
- Online certification: if online multiplayer, must support Nintendo's online service infrastructure. Peer-to-peer not allowed (Nintendo provides servers).

## Common Pitfalls

**1. Late Certification Attempt**
- *Symptom*: Game fails certification. Rework required. Delayed release by 3 weeks.
- *Cause*: Left certification testing to last week before intended release.
- *Solution*: Certification readiness check 6 weeks before intended release. Time for rework.

**2. Platform-Specific Crash**
- *Symptom*: Game passes dev testing (PC/personal console). Fails certification (specific console build has random crash).
- *Cause*: Console-specific memory corruption (uninitialized variable, different struct packing between platforms).
- *Solution*: Test every build on actual console hardware during development, not just dev kit emulation.

**3. Certification Ambiguity**
- *Symptom*: Certification request rejected. Feedback vague ("Does not meet performance standard").
- *Cause*: Certification submission incomplete (missing profiling data, frame time logs).
- *Solution*: Submit certification kit with full profiling data (frame time histogram, crash logs, load time measurements). More data = clearer why you pass.

## Tools & Workflow

**Certification Submissions**: Use platform-specific submission tools (PlayStation Network upload tool, Xbox Partner Center). Include build, profiling artifacts, test results.

**Feedback Loop**: If rejection, request detailed feedback. Sony/Microsoft certification engineers provide specific frame times/issues. Address exact issues, resubmit.

## Related Topics

- [29.3 Workflow Optimization](29-03-Workflow-Optimization.md) — Pre-certification readiness
- [05 Performance Optimization](../chapters/05-Performance-Optimization.md) — Techniques to meet requirements
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Tools for measuring

---

[← Previous: 29.3 Workflow Optimization](29-03-Workflow-Optimization.md) | [Next: 30.1 Linear Algebra →](../subjects/30-01-Linear-Algebra.md)
