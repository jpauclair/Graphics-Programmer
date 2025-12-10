# 21.3 Input Latency

[← Back to Chapter 21](../chapters/21-Frame-Management-And-Synchronization.md) | [Main Index](../README.md)

Input latency = delay from button press to screen update (motion-to-photon latency in VR). Total latency = input sampling (8-16ms = polling rate), CPU processing (game logic + rendering commands = 10-20ms), GPU rendering (16ms at 60 FPS), display lag (monitor response + buffering = 10-50ms). Target: <50ms (competitive gaming = responsive feel), <20ms (VR = avoid motion sickness), <100ms (casual = acceptable). Optimization: reduce buffering (triple buffering → double = one frame less lag), early input sampling (poll at frame start = fresher data), prediction (client-side = compensate network lag).

---

## Overview

Latency Sources: input device polling (USB = 8ms/125Hz or 1ms/1000Hz gaming mouse), OS input queue (Windows = 5-10ms), Unity Input.GetButton (sampled once per frame = up to 16ms stale at 60 FPS), game logic processing (Update → LateUpdate → rendering = 5-10ms), GPU rendering (render pipeline = 16ms at 60 FPS), frame buffering (triple buffer = 3 frames ahead = 48ms lag at 60 FPS), display (panel response time = 1-10ms + input lag = 5-30ms). Total: 60-150ms typical desktop, 20-40ms optimized competitive setup.

VR Latency: critical for comfort (>20ms motion-to-photon = nausea). Breakdown: sensor sampling (HMD tracking = 1-2ms), prediction (SDK predicts head position at display time = 5-10ms ahead), rendering (11ms at 90 FPS), display (OLED = 3-5ms, LCD = 10-20ms), total target <20ms. Optimizations: asynchronous timewarp (GPU reprojects last frame = fill missed frames), late latching (update head pose just before GPU submission = fresher tracking).

## Key Concepts

- **Input Polling Rate**: how often hardware sampled. Standard mouse/keyboard: 125 Hz (8ms = polled every 8ms), gaming peripherals: 1000 Hz (1ms = polled every millisecond). Console controllers: 250 Hz (4ms = PS5 DualSense), VR controllers: 500-1000 Hz (1-2ms = high precision). Unity: Input.GetButton samples once per frame (Update = up to 16ms stale if polled at frame start, input arrived just after). New Input System: polling more frequent (background thread = lower latency).
- **Frame Buffering**: queued frames ahead of GPU. Single buffering: CPU waits for GPU (no parallelism = lower FPS but lowest latency), Double buffering: 1 frame ahead (standard = good balance, ~16ms latency), Triple buffering: 2 frames ahead (smoothest FPS = highest latency, ~32ms extra lag). Unity: QualitySettings.maxQueuedFrames (0-2, default 2 = triple buffer). Competitive games: set to 0 (single buffer = lowest latency, accept lower FPS).
- **CPU-GPU Lag**: frames in flight between CPU and GPU. Measured: Unity Profiler → GPU (check frame offset = if GPU frame N, CPU frame N+2 = 2 frames lag). Reduce: lower maxQueuedFrames (accept GPU idle time = lower throughput but lower latency), VSync off (no blocking = GPU starts next frame immediately, but tearing).
- **Display Lag**: monitor delay (input lag = time from signal received to pixels lit). Gaming monitors: 1-5ms (TN panels = fastest), standard monitors: 10-30ms (IPS/VA = slower), TVs: 30-100ms (image processing = "Game Mode" reduces). Measure: external tools (Leo Bodnar input lag tester = high-speed camera), NVIDIA Reflex Latency Analyzer (G-Sync monitors = measure system latency). Unity can't control display lag (hardware dependent = player's monitor).
- **NVIDIA Reflex**: low-latency rendering mode (reduces CPU-GPU lag). Technique: just-in-time CPU submission (delay CPU work = submit closer to GPU execution, reduce queued frames). Unity: HDRP Reflex integration (Package Manager → NVIDIA Reflex), URP experimental. Benefits: 10-30ms latency reduction (competitive FPS = measurable advantage), minimal FPS impact (smart scheduling = maintain throughput). Platforms: PC only (NVIDIA GPUs = GeForce 900+).
- **VR Asynchronous Timewarp (ATW)**: compositor reprojects last frame if new frame missed. Process: GPU takes last rendered frame (N-1), applies latest head rotation (from tracking), displays reprojected result (rotation-only = cheap). Benefits: maintain 90 FPS (even if app renders 45 FPS = compositor fills in), reduce motion sickness (head rotation always responsive = positional lag only). Platforms: Oculus/Meta Quest (built-in), SteamVR (motion smoothing), PSVR2 (reprojection).

## Best Practices

**Reduce Input Lag**:
- New Input System: use instead of legacy Input Manager (background polling = lower latency).
- Early polling: call Input.GetButton at start of Update (immediately after poll = freshest data).
- Direct input: platform-specific APIs (XInput on Windows = bypass Unity abstraction, save 2-5ms).

**Optimize Frame Pipeline**:
- maxQueuedFrames: set to 0 or 1 (competitive games = lowest latency), default 2 (casual = smoothest).
- VSync: disable for competitive (accept tearing = lower latency), enable for casual (smooth = no tearing).
- Target frame rate: maintain solid 60 FPS (missed frames = latency spikes, consistent better than variable).

**VR Optimization**:
- Maintain 90 FPS: never miss frame (ATW helps but not perfect = positional judder).
- Single Pass Instanced: render both eyes simultaneously (lower latency than multi-pass).
- Late latching: use XR SDK (OpenXR = updates head pose just before GPU submit).

**Platform-Specific**:
- **PC**: NVIDIA Reflex (supported GPUs = enable for competitive), AMD Anti-Lag (similar, Radeon GPUs).
- **Console**: fixed frame pacing (120 Hz mode on PS5/Xbox = 8.3ms latency per frame vs 16ms at 60 Hz).
- **VR**: ATW mandatory (built-in = maintain 90 FPS perceived), optimize for 11ms budget (90 FPS).

## Common Pitfalls

**Triple Buffering on Competitive Games**: developer leaves maxQueuedFrames = 2 (default). Symptom: 50ms+ input lag (3 frames queued = 48ms at 60 FPS). Solution: QualitySettings.maxQueuedFrames = 0 (single buffer = 16ms latency, accept lower FPS if GPU can't keep up).

**Polling Input Late in Frame**: developer calls Input.GetButton in LateUpdate. Symptom: extra 10-15ms lag (input sampled at Update, not used until LateUpdate = stale). Solution: poll in Update (early as possible), cache result (use throughout frame).

**VSync On for Competitive**: developer enables VSync (smooth visuals = no tearing). Symptom: input lag (VSync forces wait = GPU idle until next vsync interval, extra 16ms). Solution: VSync off (competitive = low latency priority), or G-Sync/FreeSync (variable refresh = no tearing, low latency).

## Tools & Workflow

**Unity Input System Setup** (Low Latency):
```csharp
// Package Manager → Input System
using UnityEngine.InputSystem;

void Update()
{
    // Poll immediately (early in Update = freshest data)
    if (Keyboard.current.wKey.wasPressedThisFrame)
    {
        MoveForward(); // Immediate response
    }
}

// Configure Input System settings
// Edit → Project Settings → Input System Package
// Update Mode: Process Events In Dynamic Update (lower latency than Fixed Update)
```

**Reduce Frame Buffering**:
```csharp
void Start()
{
    // Single buffering (lowest latency, may reduce FPS)
    QualitySettings.maxQueuedFrames = 0; // 0 = single buffer, 1 = double, 2 = triple (default)
    
    // Disable VSync (competitive games)
    QualitySettings.vSyncCount = 0; // 0 = off, 1 = every VBlank (60 Hz), 2 = every other (30 Hz)
    
    // Set target frame rate
    Application.targetFrameRate = -1; // -1 = unlimited (or 60, 120, 144 for cap)
}
```

**NVIDIA Reflex Integration** (HDRP):
```csharp
// Package Manager → NVIDIA Reflex
using UnityEngine.Rendering.HighDefinition;

void Start()
{
    // Enable Reflex (low latency mode)
    var volume = FindObjectOfType<Volume>();
    if (volume.profile.TryGet<Reflex>(out var reflex))
    {
        reflex.mode.value = ReflexMode.LowLatency; // or LowLatencyBoost (higher power = lower latency)
    }
}
```

**VR Late Latching** (OpenXR):
```csharp
// XR Plugin Management → OpenXR
// Late latching enabled by default (updates head pose just before submit)

void LateUpdate()
{
    // Camera position updated here (latest tracking data)
    // Unity XR SDK automatically applies late latching (no manual code needed)
}
```

**Measure Input Latency** (Manual):
```csharp
// High-speed camera method
1. Display white square when button pressed
2. Film screen + hand at 240+ FPS (slow-motion camera)
3. Count frames: button press → square appears
4. Calculate: frame_count / 240 = latency in seconds

// NVIDIA Reflex Latency Analyzer (hardware)
1. Connect mouse to Reflex analyzer (G-Sync monitor)
2. Play game (Reflex SDK enabled)
3. Overlay shows: PC Latency (system), Game Latency (render), Display Latency (monitor)
```

**Input Prediction** (Network Games):
```csharp
// Client-side prediction (compensate network lag)
void Update()
{
    // Local prediction (immediate response)
    Vector3 inputDir = new Vector3(Input.GetAxis("Horizontal"), 0, Input.GetAxis("Vertical"));
    transform.position += inputDir * speed * Time.deltaTime;
    
    // Send input to server
    SendInputToServer(inputDir);
}

// Server reconciliation
void OnServerUpdate(Vector3 serverPos)
{
    // If server position differs significantly, correct
    if (Vector3.Distance(transform.position, serverPos) > 0.5f)
    {
        transform.position = Vector3.Lerp(transform.position, serverPos, 0.1f); // Smooth correction
    }
}
```

**VSync vs Tearing Trade-off**:
```csharp
// Competitive: prioritize latency
QualitySettings.vSyncCount = 0; // Off = lowest latency, accept tearing

// Casual: prioritize visual quality
QualitySettings.vSyncCount = 1; // On = smooth, no tearing, +16ms latency

// G-Sync/FreeSync: best of both (variable refresh rate)
// Enable in monitor settings (no Unity code needed)
// GPU drives refresh rate = no tearing, low latency
```

**VR Frame Timing Budget**:
```csharp
// 90 FPS = 11.1ms per frame budget
void Update()
{
    // Profile critical path
    float startTime = Time.realtimeSinceStartup;
    
    DoGameLogic(); // Budget: 2-3ms
    
    float elapsed = (Time.realtimeSinceStartup - startTime) * 1000f;
    if (elapsed > 3.0f)
        Debug.LogWarning($"Game logic exceeded budget: {elapsed}ms");
}

// GPU budget: 8ms rendering (leave 3ms for compositor ATW)
```

## Related Topics

- [21.1 Frame Pacing Fundamentals](21-01-Frame-Pacing-Fundamentals.md) - Consistent frame delivery
- [21.2 Console Frame Pacing](21-02-Console-Frame-Pacing.md) - Platform-specific pacing
- [27.2 Virtual Reality Rendering](27-02-Virtual-Reality-Rendering.md) - VR latency requirements
- [32.4 Cloud And Streaming Graphics](32-04-Cloud-And-Streaming-Graphics.md) - Network latency

---

[← Previous: 21.2 Console Frame Pacing](21-02-Console-Frame-Pacing.md) | [Next: 21.4 GPU Synchronization →](21-04-GPU-Synchronization.md)
