# 28.4 Analytics and Telemetry

[← Back to Chapter 28](../chapters/28-Instrumentation-And-Telemetry.md) | [Main Index](../README.md)

Analytics and telemetry collect gameplay, performance, and user data from shipped games to identify optimization opportunities. Unlike development profiling, telemetry runs in production builds with minimal overhead, aggregates data across thousands of players, and identifies platform-specific issues. This section covers telemetry architecture, privacy-compliant data collection, data aggregation, and analytics visualization.

---

## Overview

Telemetry value: shipped game runs on unknown hardware. Profiler can't connect to all devices. Solution: game logs telemetry (frame drops, crashes, memory peaks) to server. Aggregate data reveals patterns: "95% of crashes happen on 2GB RAM devices", "frame rate drops 20% on maps with >50k objects". Data informs optimization priorities.

Telemetry overhead: minimal (1-2% GPU/CPU). Typical data: frame time, crashes, memory, device info, level info. Privacy critical: disclose collection, allow opt-out (GDPR/CCPA required). No PII (user IDs, IP addresses, game saves, input logs).

## Key Concepts

- **Telemetry Payload**: Lightweight data sent to server. Typical: [session_id, timestamp, device_info, frame_time, crash, memory]. Binary format, compressed. Size: 100-500B per event.

- **Event-Based Logging**: Only log significant events (crashes, frame drops >50ms, memory >90%). Reduces bandwidth 100x. Example: log frame drop once with duration.

- **Batching**: Game accumulates events (10-100), sends batch every 60 seconds. Reduces network overhead, compression more effective.

- **Privacy**: Disable PII. No IP addresses, user IDs, save data, input logs. Only aggregate metrics and device info. GDPR/CCPA: opt-in, opt-out easy, data deletion available.

- **Differential Privacy**: Aggregate with statistical noise (individual user data can't be inferred). Example: bucketed frame times (0-10ms, 10-20ms) instead of exact values.

- **Server-Side Aggregation**: Raw events dumped to database. Aggregate jobs compute statistics (90th percentile frame time, crash frequency by device).

## Best Practices

**Telemetry Data Collection (C#):**

```csharp
public struct TelemetryEvent
{
    public uint SessionID;        // 4B unique game session
    public uint DeviceID;         // 4B hash of GPU+CPU+RAM (no PII)
    public ushort FrameTime;      // 2B frame time in ms
    public byte GPUVendor;        // 1B (0=NVIDIA, 1=AMD, 2=Intel)
    public byte MemoryPercent;    // 1B (0-100% of budget)
    public uint MapHash;          // 4B hash of current level
}

public class TelemetrySystem : MonoBehaviour
{
    private uint m_SessionID = (uint)Random.Range(0, int.MaxValue);
    private Queue<TelemetryEvent> m_EventBuffer = new Queue<TelemetryEvent>();
    private float m_BatchTimer = 0;
    private const float BATCH_INTERVAL = 60.0f;
    private const float FRAME_SPIKE_THRESHOLD = 33.33f;
    
    void Update()
    {
        float frameTimeMS = Time.deltaTime * 1000.0f;
        
        // Only log frame spikes (event-based logging)
        if (frameTimeMS > FRAME_SPIKE_THRESHOLD)
        {
            TelemetryEvent evt = new TelemetryEvent
            {
                SessionID = m_SessionID,
                DeviceID = ComputeDeviceID(),
                FrameTime = (ushort)(frameTimeMS * 10),
                GPUVendor = GetGPUVendor(),
                MemoryPercent = (byte)(GetMemoryPercent() * 255),
                MapHash = GetMapHash()
            };
            m_EventBuffer.Enqueue(evt);
        }
        
        m_BatchTimer += Time.deltaTime;
        if (m_BatchTimer > BATCH_INTERVAL)
            SendBatch();
    }
}
```

**Crash Reporting Integration:**

```csharp
public class CrashReporter
{
    public static void ReportException(Exception ex)
    {
        // Log to telemetry system
        TelemetryEvent evt = new TelemetryEvent
        {
            SessionID = m_SessionID,
            FrameTime = 0xFFFF,  // Signal crash
            Stacktrace = ex.StackTrace.GetHashCode()
        };
        m_EventBuffer.Enqueue(evt);
        SendBatchImmediate();  // Don't wait 60s for crash
    }
}

// Global exception handler
Application.logMessageReceived += (msg, trace, type) => 
{
    if (type == LogType.Exception)
        CrashReporter.ReportException(new Exception(msg));
};
```

**Server-Side Aggregation (Python):**

```python
import pandas as pd
from datetime import datetime, timedelta

# Load events from database
events_df = pd.read_csv('telemetry_events.csv')

# Compute percentiles by device
frame_times = events_df.groupby('device_vendor')['frame_time_ms'].quantile([0.50, 0.95, 0.99])
print(frame_times)
# Output:
# device_vendor
# AMD       0.50    16.5
#           0.95    45.2
#           0.99    78.9
# NVIDIA    0.50    14.2
#           0.95    32.1
#           0.99    51.3

# Crash frequency by device
crashes_per_device = events_df[events_df['frame_time'] == 0xFFFF].groupby('device_vendor').size()
total_sessions = events_df.groupby('device_vendor').size()
crash_rate = (crashes_per_device / total_sessions * 100).fillna(0)
print(crash_rate)
# Output:
# device_vendor
# AMD       2.3%
# NVIDIA    0.8%

# Alert if p99 exceeds threshold (>50ms)
if frame_times.loc['AMD', 0.99] > 50.0:
    send_alert("AMD GPU p99 frame time critical: 78.9ms")
```

**Privacy & GDPR Compliance:**

- No IP addresses or user IDs. Device ID = hash(GPU + CPU + RAM), not hardware serial.
- Clear privacy policy: "We collect frame times and device info to improve performance. Opt-out: Settings > Privacy > Telemetry."
- Data retention: delete events older than 90 days. Honor user data deletion requests within 30 days.
- Audit: verify no PII in logs before shipping.

## Platform-Specific Guidance

**PC (Steam/Windows):**
- Steam built-in crash reporting (CrashPad). Custom telemetry: HTTPS required, certificate pinning recommended.
- Network reliability: game may disconnect (WiFi dropout). Queue events, retry on reconnect.

**Console (PS5/Xbox Series X):**
- Platform-provided telemetry (PSN analytics, Xbox Live services). Use official APIs.
- Custom telemetry: HTTPS required. Network latency: console dev kit <10ms, production <50ms typical.
- Rate limiting: server accepts max 100 events/second per session to prevent abuse.

**Mobile (Android/iOS):**
- Google Play Services / Apple Analytics: built-in crash reporting (preferred).
- Custom telemetry: ensure HTTPS. Support cellular data restrictions (WiFi only option).
- Bandwidth concern: 100 events/day max. Use differential privacy to reduce resolution.

## Common Pitfalls

**1. PII in Telemetry (Privacy Violation)**
- *Symptom*: IP address or user ID logged. GDPR violation, lawsuit risk.
- *Cause*: Logging raw data without filtering.
- *Solution*: Hash IP, strip user IDs. Audit before shipping. Automated PII detection in CI.

**2. Telemetry Lag (Stale Data)**
- *Symptom*: Dashboard shows frame times from 2 weeks ago, not latest optimization.
- *Cause*: Data processing pipeline slow (ETL batch job runs once/week).
- *Solution*: Real-time aggregation (streaming pipeline, Apache Kafka). Or daily batch job.

**3. Biased Sampling**
- *Symptom*: Telemetry shows 99% stable, but forum full of stutter complaints.
- *Cause*: Players with crashes disable telemetry. Only healthy players report.
- *Solution*: Make opt-out hard. Or use differential privacy to weight crash bias. Crash reporting auto-enabled.

**4. Server Cost Explosion**
- *Symptom*: Every event 10KB data. Million events/day = 10TB storage. $100k/month cloud costs.
- *Cause*: No limits on event size or frequency. Logging every frame, not spikes.
- *Solution*: Event-based logging (only significant events). Cap payload (500B max). Gzip compression (3:1 ratio). Archive old data to cold storage.

## Tools & Workflow

**Analytics Dashboard (Grafana/Kibana):**
- Real-time graphs: frame times, crash rates, device distribution.
- SQL: "SELECT percentile_disc(0.99) WITHIN GROUP (ORDER BY frame_time) FROM frame_times WHERE device='NVIDIA' AND timestamp > NOW() - INTERVAL 7 DAY"
- Alerts: "If p99 > 50ms for >5% of players, alert Slack channel."
- Custom dashboards: memory breakdown by level, crash hotspots by platform.

## Related Topics

- [28.3 Remote Profiling](28-03-Remote-Profiling.md) — Real-time profiling on individual machines
- [28.1 Performance Metrics](28-01-Performance-Metrics.md) — Metrics collected by telemetry
- [06 Bottleneck Identification](../chapters/06-Bottleneck-Identification.md) — Using telemetry to identify issues
- [29 Case Studies](../chapters/29-Case-Studies-And-Best-Practices.md) — Real-world telemetry examples

---

[← Previous: 28.3 Remote Profiling](28-03-Remote-Profiling.md) | [Next: 29.1 AAA Game Examples →](../subjects/29-01-AAA-Game-Examples.md)
