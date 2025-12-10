# 24.4 Mobile-Specific Features

[← Back to Chapter 24](../chapters/24-Mobile-And-Cross-Platform.md) | [Main Index](../README.md)

Mobile-specific features leverage device hardware: touch (2-10 simultaneous fingers, 60-120Hz polling), sensors (accelerometer = tilt, gyroscope = rotation, GPS = location), camera (video + AR), microphone (voice input). Optimization: touch polling expensive (CPU work = every frame = power drain), batch polling (every 16ms = 60Hz sufficient), sensor fusion (combine multiple sensors = better accuracy, lower polling). AR = emerging feature (ARCore Android, ARKit iOS = 3D tracking, occlusion).

---

## Overview

Touch Input: polling interrupts CPU, input latency = critical (player expects <100ms response to tap). Methods: (1) event-driven (only process when touch occurs = efficient), (2) polling every frame (consistent = frame sync). Unity Input System = event-driven (best). Hardware polls at 60-120Hz (iPhone 6+ = 120Hz, Android flagship = 120Hz, budget = 60Hz), software can request higher (not always available). Gesture recognition = standard (swipe, pinch, long-press = multi-touch combinations).

Sensors: Accelerometer (gravity + motion = 3 axis), Gyroscope (rotation = 3 axis angular), Compass (magnetic field = direction), GPS (location = low accuracy outdoors ~5m). Sensor polling drains battery (accelerometer active = always on = background processes consume power). Solution: enable only when needed (start on level load, disable on menu = save power), low polling rate (50 Hz sufficient for accelerometer tilt, higher = diminishing returns).

Adaptive Rendering: adjust rendering based on device state (screen on/off, app in foreground/background, low power mode). OnApplicationPause(true) = backgrounded (pause rendering, reduce CPU/GPU load), OnApplicationPause(false) = resumed (resume). SystemInfo.isHeadless = VR, can optimize accordingly.

## Key Concepts

- **Touch Input Events**: Input.GetTouch(index) = returns Touch struct (position, phase, fingerId, pressure if supported). Phases: Began (first contact), Moved (drag), Stationary (held), Ended (release), Canceled (interrupted). Multi-touch = iterate all fingers, check fingerId = distinguish fingers across frames (finger 0, then 1, etc.). Event-driven = OnPointerDown/OnPointerUp (EventSystem = responds to touch events = UI-friendly). Latency = device hardware poll rate + OS processing + app processing = typical 16-20ms (60Hz).
- **Gesture Recognition**: Swipe = detect finger moved >100px in one direction (check Touch.position delta across frames). Pinch = two fingers simultaneously, measure distance change (distance = magnitude(finger0.pos - finger1.pos), pinch multiplier = new_distance / old_distance). Long-press = touch held >500ms = flag. Double-tap = two Began->Ended within 300ms. Library support = UnityEngine.EventSystems.EventTrigger or third-party (Rewired = gamepad/touch unified).
- **Accelerometer & Gyroscope**: Input.acceleration = Vector3 (x, y, z gravity + motion). Gyroscope = Input.gyro.rotationRate. Sensor fusion = combine inputs (accelerometer = coarse, gyroscope = fine = better response). Calibration = user must hold device upright on startup (store baseline = subtract from readings = true tilt). Polling overhead = enable only when needed (e.g., steering game = enable on race start, disable on menu).
- **GPS and Location**: Input.location (latitude, longitude, altitude, accuracy). Mobile OS prompts user permission (privacy = requires explicit allow). Low accuracy (±5m typical urban, ±20m indoors). Location services battery expensive (ongoing scanning = drain). Solution: coarse location (cell tower = faster lower battery) or GPS refresh interval (update every 10m movement not every frame).
- **App Lifecycle Optimization**: OnApplicationPause(isPaused) callback = foreground/background state. When paused (background = disable rendering, disable sensors, disable CPU work). When resumed = re-enable. Audio continues playing (depends on game), rendering can pause (not necessary, Profiler shows GPU idle). Battery optimization = pause physics simulation, input processing, animations = massive power savings.

## Best Practices

**Touch Input**:
- Use event system (Input System new = EventTrigger, OnPointerDown callbacks = clean, responsive). Avoid polling every frame (expensive).
- Multi-touch support = iterate all active fingers, track by fingerId (robust across frames, handles finger liftoff).
- Gesture recognition = library (Rewired, com.unity.inputsystem, or custom), test on varied devices (latency differs).
- Touch UI buttons = extend EventTrigger (OnPointerDown highlight, OnPointerUp execute action = responsive visual feedback).

**Sensor Usage**:
- Enable only when needed (steering game = accelerometer on race start, disabled on menu = battery savings ~30%).
- Calibration critical (on first use or app startup = store baseline, subtract from readings = true device tilt).
- Sensor fusion (combine accelerometer + gyroscope = smoother control, less drift than gyroscope alone).
- Polling rate = 50-100 Hz sufficient (higher = negligible improvement, battery drain).

**Adaptive Rendering**:
- OnApplicationPause(true) = disable rendering thread (even if background = save power). Set `Application.runInBackground = false;` = pauses game when backgrounded.
- Lower frame rate when backgrounded (`Application.targetFrameRate = 10;` = minimal power drain).
- Disable expensive systems (animations, physics, particle effects) when paused.

**Platform-Specific**:
- **iOS**: EventSystem = respects Safe Area (notch, home indicator) = automatic. iPhone haptics available (force feedback on taps = premium feel). Native notifications = local/remote push.
- **Android**: EventSystem same. Vibration available (Handheld.Vibrate()). Back button = hardware Navigation Back. Permissions required (Input.location = user prompt).

## Common Pitfalls

**Touch Input Polling Every Frame**: Developer polls Input.GetTouch(0) every Update() = CPU busy every frame. Symptom: high CPU usage (20-30% CPU for input alone), battery drain. Solution: event-driven (EventTrigger), or batch input processing (poll once per frame = 16ms minimum, sufficient for 60 FPS).

**Sensor Polling Always Enabled**: Developer enables accelerometer at startup, never disables. Menu screen = gravity readings still polled every frame = battery drain. Symptom: battery loss 10%+ per hour idle in menu (sensor active). Solution: disable when not needed (Input.gyro.enabled = false).

**Ignoring Permissions**: Developer calls Input.location without requesting permission first. App hangs or crashes (Android/iOS require explicit user permission). Symptom: app stalls on location access, user denies = location null. Solution: check permission first (LocationService.isEnabledByUser), request if needed (platform-specific request flow).

**Multi-touch Not Handled**: Developer assumes single touch (Input.GetMouseButtonDown), ships on phone. Two-finger tap breaks (second finger ignored = unresponsive). Symptom: pinch gesture doesn't work, long-press fails. Solution: iterate Input.GetTouch(i), handle all active fingers, distinguish by fingerId.

## Tools & Workflow

**Touch Input Testing**: Play on device (emulator input imprecise). Test scenarios: single tap, swipe (fast + slow), pinch (zoom), long-press (hold >500ms), multi-finger (two simultaneous taps). Frame rate = expect touch latency 16-33ms (one frame delay acceptable).

**Sensor Calibration Code**:
```csharp
private Vector3 accelerometerBaseline;

void CalibrateAccelerometer() {
    accelerometerBaseline = Input.acceleration;
}

Vector3 GetTiltOffset() {
    return Input.acceleration - accelerometerBaseline;
}
```

**Gesture Recognition Library** (Input System):
```csharp
void OnEnable() {
    touchAction.Enable();
}

void Update() {
    Touch touch = Input.GetTouch(0);
    if (touch.phase == TouchPhase.Moved) {
        float swipeDistance = touch.deltaPosition.magnitude;
        if (swipeDistance > 100f) HandleSwipe();
    }
}
```

**Location Service Request** (Android/iOS):
```csharp
void StartLocationService() {
    if (!Input.location.isEnabledByUser) {
        Debug.Log("Location disabled by user");
        return;
    }
    Input.location.Start();
}

void Update() {
    if (Input.location.status == LocationServiceStatus.Running) {
        float lat = Input.location.lastData.latitude;
        float lon = Input.location.lastData.longitude;
    }
}
```

**App Lifecycle Optimization**:
```csharp
void OnApplicationPause(bool isPaused) {
    if (isPaused) {
        Physics.autoSimulation = false; // Disable physics
        Input.gyro.enabled = false;     // Disable sensors
        Application.targetFrameRate = 10; // Minimal frame rate
    } else {
        Physics.autoSimulation = true;
        Application.targetFrameRate = 60;
    }
}
```

## Related Topics

- [24.1 Mobile GPU Architecture](24-01-Mobile-GPU-Architecture.md) - Hardware constraints
- [24.2 Mobile Optimization](24-02-Mobile-Optimization.md) - Battery & thermal management
- [24.3 Scalability Systems](24-03-Scalability-Systems.md) - Quality tiers
- [08.3 Input System](08-03-Input-System.md) - Input handling architecture

---

[Previous: 24.3 Scalability Systems](24-03-Scalability-Systems.md) | [Next: Chapter 25 →](../chapters/25-Build-Pipeline-And-Optimization.md)
