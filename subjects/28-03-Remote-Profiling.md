# 28.3 Remote Profiling

[← Back to Chapter 28](../chapters/28-Instrumentation-And-Telemetry.md) | [Main Index](../README.md)

Remote profiling streams performance data from running game to development machine for real-time analysis. Enables debugging shipped builds (QA testing, live player machines) without modifying code. Typical architecture: game sends profiling data over network (TCP/UDP), PC receives and visualizes (timeline, graphs, metrics). This section covers data streaming protocols, bandwidth optimization, visualization tools, and platform-specific networking approaches (console networking, mobile Wi-Fi).

---

## Overview

Remote profiling solves: QA finds bug on PS5 dev kit (can't run PC profiler directly). Developer streams profiling data from PS5 to PC, analyzes in real-time. Alternative: ship custom telemetry logging to file, ship game, retrieve logs post-play (post-mortem analysis, slower feedback loop). Remote profiling is live, enabling developer to ask "change this setting, what happens?" and see instant results.

Bandwidth constraint: streaming all data (every GPU timestamp, every draw call) = 100+ MB/s (overwhelming network). Solution: compress data, sample selectively (e.g., log every 10th frame, not every frame), or aggregate on client (send summaries instead of raw data).

Networking: typically localhost for console dev (dev kit <-> PC via Ethernet, no internet latency), or Wi-Fi for mobile. Latency 10-50ms acceptable for profiling (no real-time requirement like online gaming). Bandwidth 1-10 MB/s typical (easily achievable on Ethernet/5GHz Wi-Fi).

## Key Concepts

- **Streaming Protocol**: Client (game) sends profiling data packets to PC (server). Format: binary (compact, ~1 byte per metric) vs JSON (human-readable, 10+ bytes per metric). Binary typical for production. Data compressed (zlib reduces 10:1). Packet format: [timestamp, CPU frame time, GPU frame time, draw calls, memory, custom metrics].

- **Selective Logging**: Log only data of interest (frame spikes, memory peaks, etc.), not all frames. Example: log if frame time > 20ms, or memory > 90% budget. Reduces bandwidth 10-100x.

- **Data Aggregation**: Game summarizes data locally before sending. Example: instead of sending 10k draw-call timings per frame, send [count=10k, total_ms=15, avg_ms=0.0015, max_ms=0.005]. Reduces bandwidth 100x.

- **Network Buffering**: Game buffers profiling data in memory (ring buffer, 1MB max), sends once per 100ms (10 Hz update rate). Reduces packet overhead (many small packets = overhead, fewer large packets = efficient).

- **Authentication**: Production builds require auth to enable profiling (prevent cheating). Typical: dev machine sends password/key, game validates before starting data stream. Console SDKs provide auth (Xbox Live, PlayStation Network).

## Best Practices

**Binary Streaming Protocol:**
```csharp
public struct ProfilerPacket
{
    public float CPUTime;        // 4B
    public float GPUTime;        // 4B
    public int DrawCalls;        // 4B
    public int TriangleCount;    // 4B
    public int VRAMUsageMB;      // 4B
    public float FrameRate;      // 4B
    
    public byte[] Serialize()
    {
        byte[] buffer = new byte[24];  // 6 floats/ints * 4B
        BitConverter.GetBytes(CPUTime).CopyTo(buffer, 0);
        BitConverter.GetBytes(GPUTime).CopyTo(buffer, 4);
        BitConverter.GetBytes(DrawCalls).CopyTo(buffer, 8);
        BitConverter.GetBytes(TriangleCount).CopyTo(buffer, 12);
        BitConverter.GetBytes(VRAMUsageMB).CopyTo(buffer, 16);
        BitConverter.GetBytes(FrameRate).CopyTo(buffer, 20);
        return buffer;
    }
}

public class RemoteProfiler : MonoBehaviour
{
    private TcpClient m_Client;
    private Queue<ProfilerPacket> m_DataBuffer = new Queue<ProfilerPacket>();
    
    void OnConnected()
    {
        m_Client = new TcpClient("localhost", 5555);
    }
    
    void OnFrameComplete(float cpuTime, float gpuTime, int draws, int tris, int vram)
    {
        ProfilerPacket packet = new ProfilerPacket
        {
            CPUTime = cpuTime,
            GPUTime = gpuTime,
            DrawCalls = draws,
            TriangleCount = tris,
            VRAMUsageMB = vram,
            FrameRate = 1.0f / Time.deltaTime
        };
        
        m_DataBuffer.Enqueue(packet);
        
        if (m_DataBuffer.Count >= 100)  // Send batch every 100 frames
            SendBatch();
    }
    
    void SendBatch()
    {
        int packetCount = m_DataBuffer.Count;
        byte[] buffer = new byte[4 + packetCount * 24];  // 4B for count, then packets
        BitConverter.GetBytes(packetCount).CopyTo(buffer, 0);
        
        int offset = 4;
        while (m_DataBuffer.Count > 0)
        {
            var packet = m_DataBuffer.Dequeue();
            packet.Serialize().CopyTo(buffer, offset);
            offset += 24;
        }
        
        m_Client.GetStream().Write(buffer, 0, buffer.Length);
    }
}
```

**Selective Logging (Spike Detection):**
```csharp
public class SpikeDetector
{
    private float m_AverageFrameTime = 16.67f;  // Moving average
    private const float SPIKE_THRESHOLD = 1.5f;  // 50% above average
    
    public bool ShouldLog(float frameTime)
    {
        // Update moving average (exponential smoothing)
        m_AverageFrameTime = m_AverageFrameTime * 0.95f + frameTime * 0.05f;
        
        // Log if spike detected
        if (frameTime > m_AverageFrameTime * SPIKE_THRESHOLD)
            return true;
        
        // Also log periodically (every 100 frames) for baseline data
        if (Time.frameCount % 100 == 0)
            return true;
        
        return false;
    }
}
```

**PC Receiver (C#):**
```csharp
public class ProfilerReceiver
{
    private TcpListener m_Listener;
    private List<ProfilerPacket> m_ReceivedData = new List<ProfilerPacket>();
    
    public void StartListening(int port)
    {
        m_Listener = new TcpListener(IPAddress.Loopback, port);
        m_Listener.Start();
        
        Thread acceptThread = new Thread(() => AcceptConnections());
        acceptThread.Start();
    }
    
    void AcceptConnections()
    {
        while (true)
        {
            TcpClient client = m_Listener.AcceptTcpClient();
            Thread recvThread = new Thread(() => ReceiveData(client));
            recvThread.Start();
        }
    }
    
    void ReceiveData(TcpClient client)
    {
        using (var stream = client.GetStream())
        {
            byte[] buffer = new byte[65536];
            
            while (client.Connected)
            {
                int bytesRead = stream.Read(buffer, 0, buffer.Length);
                if (bytesRead == 0) break;
                
                int packetCount = BitConverter.ToInt32(buffer, 0);
                for (int i = 0; i < packetCount; i++)
                {
                    int offset = 4 + i * 24;
                    float cpuTime = BitConverter.ToSingle(buffer, offset);
                    float gpuTime = BitConverter.ToSingle(buffer, offset + 4);
                    int draws = BitConverter.ToInt32(buffer, offset + 8);
                    // ... parse rest of packet
                    
                    ProfilerPacket packet = new ProfilerPacket { /* ... */ };
                    m_ReceivedData.Add(packet);
                }
            }
        }
    }
}
```

## Platform-Specific Guidance

**PC / Development:**
- Localhost (127.0.0.1:5555): PC dev kit connects to PC running profiler. <1ms latency. Simplest setup.
- Custom profiler UI: ImGui/WinForms displaying real-time graphs.

**Console (PS5/Xbox Series X):**
- Dev kit networking: connected to dev network. IP address assigned, profiler connects to game's TCP server.
- Authentication: Xbox Live requires auth token. PS5 requires dev kit privilege.
- Typical: game runs on PS5 dev kit, profiler runs on PC connected to same network. Ethernet preferred (Wi-Fi can have jitter).

**Mobile (Android/iOS):**
- Wi-Fi required (USB profiling not practical for games). IP address = phone's Wi-Fi IP.
- Bandwidth limited (5-10 Mbps typical on 5GHz Wi-Fi). Aggressive compression needed.
- iOS: MDM (Mobile Device Management) or developer profile required for production builds. Android more flexible (custom APK).

## Common Pitfalls

**1. Bandwidth Saturation (Network Dropped Packets)**
- *Symptom*: Profiler receives incomplete data (missing frames), network errors, "connection reset".
- *Cause*: Streaming too much data. Raw frame-time logging every frame = [8B timestamp + 8B data] * 60 fps = 960 B/frame * 60 = 57.6 KB/s (reasonable). But if logging GPU timings per draw call (16B per draw * 1000 draws = 16KB/frame * 60 = 960 KB/s), bandwidth overrun.
- *Solution*: Reduce data granularity (aggregate instead of per-frame). Or use selective logging (only spike frames). Or sample every 10th frame. Test on target network (console Ethernet 100+ Mbps OK, mobile Wi-Fi 5-10 Mbps tight).

**2. Timestamp Synchronization Issues**
- *Symptom*: Profiler shows frame 0 at timestamp T, frame 1 at timestamp T-10ms (time goes backwards). Data appears corrupted.
- *Cause*: Game and PC clocks not synchronized. Or uint overflow (if using 32-bit timestamps, overflows every 4 seconds).
- *Solution*: Use 64-bit timestamps (milliseconds since epoch). Or frame counter (frame 0, 1, 2, ...) instead of timestamps. Verify clocks: game logs frame 0 at time X, PC receives at time Y, latency = Y-X (should be consistent 10-50ms).

**3. Profiler UI Lagging (Can't Keep Up with Incoming Data)**
- *Symptom*: PC profiler UI freezes, CPU usage spikes to 100%, dropped data.
- *Cause*: Receiving data faster than UI can render. Each packet = 1 graph update, 1000 packets/sec = 1000 updates/sec (way more than 60 fps UI can handle).
- *Solution*: Buffer incoming data on thread, render on main UI thread at fixed rate (60 fps). Discard old data if buffer grows too large (keep last 300 frames = 5 seconds).

**4. Authentication Bypass / Cheating**
- *Symptom*: Player enables profiler in shipped build, reads memory addresses, cheats (e.g., detects cheat detection).
- *Cause*: No auth, or weak auth (hardcoded password). Profiler access = potential security hole.
- *Solution*: Require cryptographic auth (HMAC-SHA256 shared secret). Or platform-level auth (Xbox Live, PlayStation Network). For mobile, use MDM (corporate) or disable entirely in shipping builds.

## Tools & Workflow

**Profiler Visualization (ImGui, PC):**
```csharp
public class ProfilerWindow
{
    private List<ProfilerPacket> m_Data = new List<ProfilerPacket>();
    
    public void Render()
    {
        ImGui.SetNextWindowSize(new System.Numerics.Vector2(800, 400), ImGuiCond.FirstUseEver);
        ImGui.Begin("Profiler");
        
        ImGui.PlotLines("Frame Time (ms)", ref m_Data.Select(d => d.CPUTime + d.GPUTime).ToArray());
        ImGui.PlotLines("Draw Calls", ref m_Data.Select(d => (float)d.DrawCalls).ToArray());
        ImGui.PlotLines("VRAM (MB)", ref m_Data.Select(d => (float)d.VRAMUsageMB).ToArray());
        
        ImGui.End();
    }
}
```

## Related Topics

- [28.2 In-Game Debug UI](28-02-In-Game-Debug-UI.md) — Local profiling alternative
- [28.1 Performance Metrics](28-01-Performance-Metrics.md) — Data collection for streaming
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Industry profiling tools (Nsight, PIX integrate with remote data)
- [26 Debugging Techniques](../chapters/26-Debugging-Techniques.md) — Live debugging strategies

---

[← Previous: 28.2 In-Game Debug UI](28-02-In-Game-Debug-UI.md) | [Next: 28.4 Analytics & Telemetry →](28-04-Analytics-And-Telemetry.md)
