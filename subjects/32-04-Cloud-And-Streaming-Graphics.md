# 32.4 Cloud and Streaming Graphics

[← Back to Chapter 32](../chapters/32-Future-Proofing.md) | [Main Index](../README.md)

Cloud and streaming graphics = render on server GPU, stream video to client (Stadia, GeForce Now, Xbox Cloud Gaming). Benefits: play AAA games on low-end devices (phone/tablet = server does rendering), no downloads (instant play = stream like Netflix). Challenges: latency (input lag = 50-150ms server roundtrip), bandwidth (4K 60 FPS = 30+ Mbps), compression artifacts (video codec = quality loss). Future: edge computing (server closer to player = lower latency), 5G (higher bandwidth = better quality), neural codecs (ML compression = less bandwidth).

---

## Overview

Cloud Gaming Architecture: player input → server (50-100ms = network latency), server renders frame (16ms = 60 FPS), encode video (H.264/H.265 = 5ms), stream to client (50-100ms = network), client decodes (5ms), display (total latency = 120-200ms). Optimization: predictive input (client predicts movement = reduce perceived lag), adaptive bitrate (reduce quality when bandwidth drops), regional servers (edge computing = closest datacenter).

Compression: H.264 (most compatible, 30 Mbps for 1080p60), H.265/HEVC (2x better = 15 Mbps for 1080p60, or 30 Mbps for 4K60), AV1 (royalty-free, better than H.265 = future codec). Hardware encoding: NVIDIA NVENC (dedicated encoder on GPU = <5ms), AMD VCE, Intel QuickSync. Unity: no direct cloud streaming (third-party = Parsec SDK, custom server).

## Key Concepts

- **Input Latency**: total delay from button press to screen update. Local gaming: 50-100ms (input polling 8ms + rendering 16ms + display 33ms). Cloud gaming: 120-250ms (add network roundtrip 50-100ms × 2 = input to server + frame to client). Acceptable: <150ms (competitive = local feel), <200ms (casual acceptable), >250ms (laggy = unplayable). Mitigation: client prediction (local simulation = immediate feedback), regional servers (reduce network distance = lower latency).
- **Adaptive Bitrate Streaming**: adjust video quality based on bandwidth (Netflix-style = start high, drop if buffering). Implementation: measure network (ping, packet loss, jitter), adjust encoder bitrate (30 Mbps → 15 Mbps = lower quality but smooth), resolution scaling (4K → 1080p = bandwidth halved). Unity: integrate with streaming SDK (custom solution = measure QualityOfService.PingTime, adjust Camera resolution).
- **Server-Side Rendering**: server runs full game (Unity headless = no display, render to texture), encodes frame (NVENC = hardware H.264), streams via WebRTC or custom protocol. Advantages: client agnostic (runs on phone/tablet/browser = same server game), high-end graphics (server RTX 4090 = render ultra settings, stream to potato device). Disadvantages: server cost ($1-2 per hour per player = expensive at scale), bandwidth (30 Mbps per player = datacenter networking).
- **Edge Computing**: servers near players (CDN-like = low-latency regional datacenters). Example: AWS Wavelength (5G edge = <20ms latency), Azure Edge Zones, Google Cloud edge. Benefits: reduced latency (server 10 miles away = 10-20ms vs 50-100ms cross-country), local compliance (data stays in region). Challenges: server distribution (need datacenters everywhere = infrastructure cost).
- **Neural Compression**: ML-based video codecs (compress better than H.265). Examples: NeuroCodec (Google, 50% better compression), NVIDIA Maxine (AI-enhanced streaming = lower bandwidth), VVC (Versatile Video Coding = H.266, 30% better than H.265). Status: experimental (not yet real-time = too slow for 60 FPS), future promise (Tensor Cores accelerate = real-time ML codec). Potential: 4K 60 FPS at 10 Mbps (vs 30 Mbps H.265 = 3x better).
- **Game Streaming Platforms**: existing services (Stadia = shut down 2023, GeForce Now = NVIDIA, xCloud = Xbox, Luna = Amazon). Architecture: Unity build (Windows/Linux server = headless or windowed), run on VM (NVIDIA GPU = T4/A10), stream via proprietary protocol. Unity considerations: no console-specific code (cloud = Windows/Linux servers), optimize for encoding (reduce complexity = faster NVENC).

## Best Practices

**Latency Optimization**:
- Regional servers: deploy close to players (US East, EU West, Asia Pacific = cover 80% of players).
- Predictive input: client-side prediction (local movement = instant response, server reconciliation = correct position).
- Frame pacing: consistent 60 FPS (variable frame time = worse perceived latency).

**Bandwidth Optimization**:
- Resolution: start at 720p or 1080p (not 4K = too much bandwidth), upscale client-side (FSR/bilinear = acceptable quality).
- Bitrate: adaptive (measure bandwidth, adjust encoder = 10-30 Mbps range).
- Codec: H.265 if client supports (2x better = half bandwidth), fallback H.264 (universal compatibility).

**Encoder Settings**:
- NVENC: low-latency preset (faster encoding = lower latency), tune for ultra-low latency (trade quality for speed).
- Keyframe interval: frequent (every 30-60 frames = faster recovery from packet loss).
- Multi-threaded: parallel encoding (if multiple GPUs = one encoder per player).

**Platform-Specific**:
- **Windows Server**: NVIDIA GPUs (T4/A10/A100 = datacenter), NVENC encoding (hardware = <5ms).
- **Linux Server**: same (Ubuntu + NVIDIA drivers), headless rendering (no X server = EGL context).
- **Client**: WebRTC (browser-based = instant play, no app download), or native app (lower latency = custom protocol).

## Common Pitfalls

**Ignoring Latency Budget**: developer assumes server rendering = acceptable (200ms+ latency). Symptom: unplayable for fast-paced games (FPS, racing = input lag obvious). Solution: measure latency (client-to-client echo = roundtrip time), optimize (regional servers, predictive input), or limit to turn-based/slow games (RPG, strategy = latency less critical).

**Fixed Bitrate**: server encodes at 30 Mbps always (ignores client bandwidth). Symptom: buffering/stuttering on slow connections (mobile = 10 Mbps), or wasted bandwidth on high connections (gigabit = could do 4K). Solution: adaptive bitrate (measure client bandwidth = QoS.PingTime, packet loss), adjust encoder (15-50 Mbps range).

**CPU Encoding**: developer uses software H.264 encoder (x264 = CPU, slow). Symptom: encoding takes 30ms per frame (60 FPS impossible = 16ms budget). Solution: hardware encoding (NVENC = <5ms, dedicated silicon), or reduce resolution (720p = 4x fewer pixels = faster CPU encoding, but still too slow for real-time).

## Tools & Workflow

**Unity Server-Side Rendering**:
```csharp
// Headless server (no display, render to RenderTexture)
void Start()
{
    Camera.main.targetTexture = new RenderTexture(1920, 1080, 24);
    // Encode targetTexture each frame → stream to client
}

void LateUpdate()
{
    // Read pixels
    RenderTexture.active = Camera.main.targetTexture;
    Texture2D frame = new Texture2D(1920, 1080, TextureFormat.RGB24, false);
    frame.ReadPixels(new Rect(0, 0, 1920, 1080), 0, 0);
    frame.Apply();
    
    // Encode to H.264 (NVENC via plugin, or FFmpeg)
    byte[] encoded = EncodeH264(frame);
    StreamToClient(encoded);
}
```

**NVENC Encoding** (Native Plugin):
```cpp
// NVIDIA Video Codec SDK
#include <nvEncodeAPI.h>

// Initialize encoder
NV_ENC_INITIALIZE_PARAMS params = {};
params.encodeWidth = 1920;
params.encodeHeight = 1080;
params.frameRateNum = 60;
params.encodeConfig.rcParams.rateControlMode = NV_ENC_PARAMS_RC_CBR; // Constant bitrate
params.encodeConfig.rcParams.averageBitRate = 15 * 1000000; // 15 Mbps

nvEncInitializeEncoder(encoder, &params);

// Encode frame
NV_ENC_PIC_PARAMS picParams = {};
picParams.inputBuffer = d3dTexture; // Unity RenderTexture as D3D11 texture
nvEncEncodePicture(encoder, &picParams); // Output: H.264 bitstream
```

**WebRTC Streaming** (JavaScript Client):
```javascript
// Client receives video stream (browser-based)
const peerConnection = new RTCPeerConnection();

peerConnection.ontrack = (event) => {
  const video = document.getElementById('gameVideo');
  video.srcObject = event.streams[0]; // Display stream
};

// Send input to server
peerConnection.createDataChannel('input');
dataChannel.send(JSON.stringify({ key: 'W', pressed: true }));
```

**Adaptive Bitrate** (Unity C#):
```csharp
float measureBandwidth = 0;

void Update()
{
    // Measure network quality
    int ping = Mathf.RoundToInt(Network.GetAveragePing() * 1000); // ms
    float packetLoss = Network.GetPacketLoss(); // 0-1
    
    // Adjust encoder bitrate
    if (ping > 150 || packetLoss > 0.05f)
        encoder.SetBitrate(10_000_000); // 10 Mbps (lower quality)
    else if (ping < 50 && packetLoss < 0.01f)
        encoder.SetBitrate(30_000_000); // 30 Mbps (high quality)
    else
        encoder.SetBitrate(15_000_000); // 15 Mbps (balanced)
}
```

**Client Prediction** (Reduce Perceived Latency):
```csharp
// Client: predict movement immediately
void Update()
{
    Vector3 inputDir = new Vector3(Input.GetAxis("Horizontal"), 0, Input.GetAxis("Vertical"));
    transform.position += inputDir * speed * Time.deltaTime; // Local prediction
    
    SendInputToServer(inputDir); // Also send to server
}

// Server: authoritative position
void OnServerPosition(Vector3 serverPos)
{
    // Reconcile: if too far off, snap to server
    if (Vector3.Distance(transform.position, serverPos) > 1.0f)
        transform.position = serverPos;
}
```

**Edge Server Deployment** (AWS Example):
```bash
# Deploy Unity Linux server to AWS Wavelength (5G edge)
1. Build Unity: Linux Dedicated Server (headless)
2. Docker: package with NVIDIA drivers, NVENC support
3. AWS: deploy to Wavelength zone (near 5G towers = low latency)
4. Auto-scaling: add servers when player count increases
5. Monitoring: CloudWatch metrics (latency, bandwidth, GPU usage)
```

## Related Topics

- [25.3 Continuous Integration](25-03-Continuous-Integration.md) - Server build automation
- [28.3 Remote Profiling](28-03-Remote-Profiling.md) - Debug cloud servers
- [32.3 Machine Learning In Graphics](32-03-Machine-Learning-In-Graphics.md) - Neural compression
- [21.2 Console Frame Pacing](21-02-Console-Frame-Pacing.md) - Consistent frame delivery

---

[← Previous: 32.3 Machine Learning In Graphics](32-03-Machine-Learning-In-Graphics.md) | [Main Index](../README.md)

**🎉 Congratulations! You have completed all 138 subjects in the Unity Graphics Optimization Handbook. 🎉**
