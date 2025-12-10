# 28.2 In-Game Debug UI

[← Back to Chapter 28](../chapters/28-Instrumentation-And-Telemetry.md) | [Main Index](../README.md)

In-game debug overlays display real-time metrics during gameplay: frame time, GPU occupancy, draw calls, memory usage, and custom application metrics. Debug UIs are essential development tools (quick feedback loop) and production diagnostic tools (remote debugging, player performance reports). This section covers UI implementation patterns, overlay architecture, performance impact minimization, and platform-specific considerations.

---

## Overview

Debug UI benefits: immediate feedback (change graphics setting, see frame time instantly), development velocity (no context switch to external profiler), and production diagnostics (QA reports "game stutters on PS5", developer attaches remote debug UI, sees VRAM spikes). Trade-offs: rendering text/graphs costs GPU time, memory, and code complexity. Production builds should disable by default (performance cost), but allow remote enable (via development console).

Typical debug UI layers: basic (frame time, draw calls, 2-4 numbers), intermediate (memory breakdown, GPU timeline, shader counts), and detailed (per-draw-call timing, instruction cache misses, network latency). Development: enable detailed. QA/Player builds: basic only, can remotely enable intermediate if needed.

## Key Concepts

- **Overlay Rendering**: Render debug UI on top of game without disrupting game rendering. Typical: after main scene render, render UI via simple quad rendering (textured quads for text, triangles for graphs). Cost: 1-5ms GPU (text rendering expensive if rendering many glyphs).

- **Frame Time Visualization**: Graph of frame times over last 300 frames (5 seconds at 60 fps). Green (0-16.67ms), yellow (16.67-33.33ms), red (>33.33ms). Visual pattern recognition: spikes every 1000 frames = GC, every 60 frames = some periodic logic, random = variable load.

- **Memory Breakdown**: Pie chart or bar chart showing VRAM allocation (textures %, meshes %, render targets %, buffers %). Total memory vs budget. Identifies bloat (90% textures = optimize textures, not meshes).

- **Draw Call Timeline**: Horizontal bar graph showing per-draw-call timing. Each bar = 1ms width = time for that draw call (relative to frame budget). Stacked bars show relative cost. Identifies heavy draws (bar taller than others).

- **Remote Debug Enable**: Production builds include debug UI code (disabled by default). Development console command (or network packet) enables UI remotely. Typical: telnet to game, type "debugui on", UI renders next frame. Cost: <1ms once per frame.

## Best Practices

**Text Rendering (Efficient):**
- Pre-baked glyph atlas: render text characters to texture at startup (128x128 atlas, 256 glyphs = 8x8 grid). VS/PS render textured quads per character. Cost: 1 glyph = 2 triangles, typical UI = 50-200 glyphs = 100-400 triangles, negligible GPU cost.
- Minimize text updates: cache text geometry (rebuild only if text changes, not every frame). Typical: frame time text = constant format string ("FPS: 60.0"), reuse same quad buffer, update texture UV only.
- Example C# text rendering:
  ```csharp
  public class DebugText
  {
      private string m_CachedText = "";
      private Mesh m_CachedMesh = null;
      
      public void UpdateText(string text, float x, float y)
      {
          if (text == m_CachedText) return;  // No change, reuse mesh
          
          m_CachedText = text;
          m_CachedMesh = BuildTextMesh(text, x, y);  // Only rebuild if text changes
      }
      
      public void Render(CommandBuffer cmd)
      {
          cmd.DrawMesh(m_CachedMesh, Matrix4x4.identity, m_DebugTextMaterial);
      }
  }
  ```

**Graph Rendering (Circular Buffer):**
```csharp
public class FrameTimeGraph
{
    private float[] m_FrameTimes = new float[300];  // 5 seconds at 60 fps
    private int m_WriteIndex = 0;
    
    public void RecordFrameTime(float timeMS)
    {
        m_FrameTimes[m_WriteIndex] = timeMS;
        m_WriteIndex = (m_WriteIndex + 1) % 300;
    }
    
    public void Render(CommandBuffer cmd, Rect screenRect)
    {
        // Render graph as line strip (300 vertices, interpolate frame times)
        Vector3[] vertices = new Vector3[300];
        for (int i = 0; i < 300; i++)
        {
            int idx = (m_WriteIndex + i) % 300;
            float x = screenRect.x + (i / 300.0f) * screenRect.width;
            float y = screenRect.y + Mathf.Clamp01(m_FrameTimes[idx] / 33.33f) * screenRect.height;
            vertices[i] = new Vector3(x, y, 0);
        }
        
        Mesh graphMesh = new Mesh();
        graphMesh.SetVertices(vertices);
        graphMesh.SetIndices(Enumerable.Range(0, 300).ToArray(), MeshTopology.LineStrip, 0);
        
        cmd.DrawMesh(graphMesh, Matrix4x4.identity, m_GraphMaterial);
    }
}
```

**Memory Breakdown Visualization:**
- Sample memory per frame: texture memory, buffer memory, RenderTexture memory, managed allocations. Total vs budget.
- Pie chart: 4 slices (textures, buffers, RT, other). Each slice = percentage of total.
- Update frequency: every 10 frames (expensive to query all memory sources every frame). Smooth display by averaging.

**Remote Debug Console:**
```csharp
public class DebugConsole
{
    private bool m_UIEnabled = false;
    
    public void ProcessCommand(string cmd)
    {
        if (cmd == "debugui on")
        {
            m_UIEnabled = true;
            Debug.Log("Debug UI enabled");
        }
        else if (cmd == "debugui off")
        {
            m_UIEnabled = false;
            Debug.Log("Debug UI disabled");
        }
        else if (cmd == "debugui graph")
        {
            m_ShowFrameTimeGraph = !m_ShowFrameTimeGraph;
        }
    }
    
    void Update()
    {
        if (m_UIEnabled)
            RenderDebugUI();
    }
    
    void RenderDebugUI()
    {
        // Render frame time, draw calls, memory, etc.
        ImGui.Text($"FPS: {1.0f / Time.deltaTime:F1}");
        ImGui.Text($"Frame Time: {Time.deltaTime * 1000.0f:F2}ms");
        ImGui.Text($"Draw Calls: {GetDrawCallCount()}");
    }
}
```

**Performance Impact Minimization:**
- Disable by default (production builds). Enable only via console command or development build.
- Throttle updates: update metrics every 5 frames (12 fps metric refresh at 60 fps game). Imperceptible delay, 5x cost reduction.
- Use efficient text rendering (glyph atlas, no string allocations per frame).
- Avoid expensive queries: GetSystemMemoryInfo() may lock, don't call every frame. Cache results, update every 1 second.

## Platform-Specific Guidance

**PC (Unity/Unreal):**
- Easy: text rendering via IMGui/Slate. Built-in metrics: frame time, draw calls, memory (Profiler window).
- Enable/disable via editor Debug menu or console (tilde key).
- Profiler integration: link debug UI to Profiler (clicking on spike in graph opens detailed Profiler data).

**Console (PS5/Xbox Series X):**
- Official dev kits: built-in debug UIs (Gnmx profiler, PIX). In-game debug overlays useful for QA.
- Implement minimal: frame time, draw calls (expensive to render complex graphs on console).
- Performance critical: console profilers are performant; in-game overlays cost 1-3ms (acceptable).

**Mobile (Android/iOS):**
- Text rendering challenge: no IMGui equivalent. Use custom text implementation (glyph atlas).
- Typical debug UI: single line of text (FPS, draw calls), optionally graph. Keep minimal.
- On-device UI expensive (GPU bound on mobile). Consider streaming to development machine instead (next section: Remote Profiling).

## Common Pitfalls

**1. String Allocation Per Frame**
- *Symptom*: Debug UI enabled, GC spikes every 1-2 seconds (frame drops), excessive allocations.
- *Cause*: Building strings per frame: `string fps = "FPS: " + (1.0f / Time.deltaTime).ToString();` causes allocation, then garbage collection.
- *Solution*: Pre-allocate strings, use string.Format with cached format string. Or StringBuilder (no GC). Typical: `Text("FPS: {0:F1}", fps);` where format string is pre-allocated.

**2. Graph Rendering Too Expensive**
- *Symptom*: Frame time graph enabled, game FPS drops from 60 to 50 fps (10% overhead).
- *Cause*: Rendering 300+ vertices per frame (line strip with many vertices). Or texture filtering slow on old mobile.
- *Solution*: Reduce graph resolution (sample every 2nd frame = 150 vertices). Or use simpler visualization (bar graph = 10 quads, much cheaper). Test on target hardware.

**3. Memory Queries Block Rendering**
- *Symptom*: Frame spikes unpredictably when debug UI queries memory.
- *Cause*: GetSystemMemoryInfo() or similar calls flush GPU pipeline (GPU->CPU sync). Blocks rendering.
- *Solution*: Cache results, update every 1 second (not every frame). Or query asynchronously (separate thread, 1-frame latency).

**4. Remote Console Not Reachable**
- *Symptom*: Can't enable debug UI on released game (no console visible).
- *Cause*: Console not networked. Or disabled in shipping build.
- *Solution*: Implement network console (UDP/TCP listener, localhost or remote). Allow enable via network packet (authentication required to prevent cheating).

## Tools & Workflow

**ImGui Integration (C#/Unity):**
```csharp
using ImGuiNET;

public class DebugUI : MonoBehaviour
{
    void OnGUI()
    {
        ImGui.SetNextWindowPos(new System.Numerics.Vector2(10, 10));
        ImGui.Begin("Debug", ImGuiWindowFlags.AlwaysAutoResize);
        
        ImGui.Text($"FPS: {1.0f / Time.deltaTime:F1}");
        ImGui.Text($"Frame Time: {Time.deltaTime * 1000.0f:F2}ms");
        ImGui.Text($"Draw Calls: {UnityEngine.Rendering.GraphicsMetrics.GetCurrentRenderingThreadIndex()}");
        ImGui.Text($"VRAM: {SystemInfo.graphicsMemorySize / 1024 / 1024}MB");
        
        ImGui.End();
    }
}
```

## Related Topics

- [28.1 Performance Metrics](28-01-Performance-Metrics.md) — Data collection for debug UI display
- [28.3 Remote Profiling](28-03-Remote-Profiling.md) — Streaming debug UI to dev machine
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Professional profiling tools vs. in-game UI
- [26 Debugging Techniques](../chapters/26-Debugging-Techniques.md) — Visual debugging and interactive tools

---

[← Previous: 28.1 Performance Metrics](28-01-Performance-Metrics.md) | [Next: 28.3 Remote Profiling →](28-03-Remote-Profiling.md)
