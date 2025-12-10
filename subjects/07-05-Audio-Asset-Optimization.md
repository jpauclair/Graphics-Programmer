# 7.5 Audio Asset Optimization

[← Back to Chapter 7](../chapters/07-Asset-Optimization.md) | [Main Index](../README.md)

Audio asset optimization reduces memory consumption, loading times, and streaming overhead through compression, format selection, and loading strategies.

---

## Overview

Audio assets consume 5-15% of memory in typical games but can create performance issues: uncompressed audio wastes memory (10MB per minute of stereo audio), synchronous loading causes hitches (100-500ms loading music track), and excessive streaming I/O competes with texture streaming. A game with 500 sound effects (average 2 seconds) and 30 music tracks (average 3 minutes) consumes 600MB uncompressed, but compresses to 60-120MB with Vorbis/ADPCM—80-90% reduction.

Audio optimization focuses on three areas: compression (Vorbis for music/ambient, ADPCM for short effects), loading strategy (streaming for music, decompress on load for effects, preload in memory for critical sounds), and quality settings (bitrate selection, sample rate reduction, mono vs stereo). Platform considerations matter: Switch and mobile have limited memory (prefer streaming), consoles have fast I/O (streaming works well), PC varies (provide quality settings).

Optimization workflow: audit audio assets via Audio Import Settings and Memory Profiler, apply compression per audio type (music = Vorbis streaming, effects = ADPCM decompress on load, UI = ADPCM in memory), set quality tiers (Low = 64kbps, High = 192kbps), and profile memory/I/O impact. Results: 70-90% memory reduction, eliminated loading hitches, and scalability across platforms.

## Key Concepts

- **Audio Compression**: Lossy compression (Vorbis, MP3, ADPCM) or lossless (FLAC, uncompressed PCM). Vorbis offers best quality/size ratio (10:1 at 128kbps). ADPCM provides fast decompression (4:1 compression) for short effects.
- **Load Type**: How Unity loads audio. Decompress On Load (decode at load time, store PCM in memory), Compressed In Memory (store compressed, decode during playback), Streaming (load chunks from disk during playback).
- **Sample Rate**: Audio frequency (22kHz, 44.1kHz, 48kHz). Higher = better quality, more memory. Reduce to 22kHz for non-critical sounds (ambient, UI) without noticeable quality loss.
- **Bitrate**: Compression quality (64kbps, 128kbps, 192kbps). Higher = better quality, larger files. 128kbps Vorbis provides excellent quality for most music. 192kbps for high-fidelity audio.
- **Mono vs Stereo**: Stereo doubles file size vs mono. Use stereo for music and positional ambience. Use mono for sound effects (3D audio system provides spatialization).

## Best Practices

**Compression Strategy:**
- Use Vorbis for music and long ambient sounds: Highest quality at moderate bitrates (128kbps = near-CD quality, ~10:1 compression). Import Settings > Load Type = Streaming, Compression Format = Vorbis.
- Use ADPCM for short sound effects: Fast decompression (HW accelerated on consoles). 4:1 compression ratio. Import Settings > Compression Format = ADPCM, Load Type = Decompress On Load.
- Use MP3 for mobile (iOS/Android): Platform-native decoders optimize MP3 playback. Better battery life than Vorbis. Import Settings > Platform Override > iOS/Android > MP3.
- Never use PCM (uncompressed) except for very short UI sounds (<0.5s, frequently played). PCM avoids decompression CPU cost but wastes memory (10x larger than Vorbis).

**Load Type Selection:**
- Stream music and long ambient tracks: Load Type = Streaming. Loads in chunks (reduces memory 10-50x), slight I/O overhead. Essential for music (3-5 min tracks = 30-50MB PCM each).
- Decompress On Load for short effects: Load Type = Decompress On Load. Decompresses at load time, stores PCM in memory. Fast playback (no runtime decompression), acceptable memory cost for short clips.
- Compressed In Memory for infrequent sounds: Load Type = Compressed In Memory. Smallest memory (stores compressed), decompression at playback (CPU cost). Good for rare sounds (death screams, special abilities).
- Preload critical sounds: Load essential sounds (gunfire, footsteps, UI) during level load. Store in memory (Decompress On Load). Avoid runtime loading for time-critical sounds.

**Sample Rate and Quality:**
- Use 44.1kHz for music: Standard CD quality. Sufficient for all music and high-fidelity sounds. Import Settings > Sample Rate Setting = Preserve Sample Rate.
- Use 22kHz for ambient and effects: Halves memory vs 44.1kHz. Quality loss negligible for non-musical sounds (footsteps, wind, UI). Import Settings > Sample Rate Setting = Override Sample Rate > 22050Hz.
- Reduce bitrate for non-critical audio: Background ambient at 64-96kbps Vorbis. Music at 128-192kbps. Footsteps/UI at 64-128kbps. Import Settings > Quality slider (0-100).
- Implement audio quality settings: Low = 22kHz, 64kbps. Medium = 44.1kHz, 96kbps. High = 44.1kHz, 192kbps. Expose to users, especially on PC (VRAM capacity varies).

**Mono vs Stereo Optimization:**
- Use mono for 3D positioned sounds: Unity's AudioSource provides spatialization (HRTF, Doppler). Mono source + 3D audio = stereo output with directionality. Stereo sources disrupt spatialization.
- Use stereo for music and 2D ambience: Music benefits from stereo mix. Background ambience (forest soundscape, city noise) uses stereo for richness.
- Force Mono on import for effects: Import Settings > Force To Mono. Reduces memory by 50% for sound effects that don't need stereo (gunshots, explosions, footsteps).
- Profile mono/stereo usage: Memory Profiler > Audio shows per-clip size. Identify stereo clips that should be mono (2x memory for no benefit).

**Memory and Streaming Management:**
- Preload audio during loading screens: Load all level-specific sounds async during load. Hides loading cost (user sees progress bar, not hitches).
- Unload unused audio: Use Addressables or AssetBundles to load/unload audio per level. Unload previous level's audio to reclaim memory.
- Limit concurrent streaming sounds: 2-4 simultaneous streams max (music + 1-3 ambient). More streams increase I/O overhead, competing with texture streaming.
- Use Audio Mixer for ducking: Lower background music/ambience volume during dialogue or combat. Reduces audio complexity, improves CPU (fewer concurrent sounds).

**Platform-Specific:**
- **PC**: Flexible (users have 8-32GB RAM). Provide quality settings (Low = streaming + low bitrate, High = decompress on load + high bitrate). Default to Medium.
- **Xbox Series X/S**: Fast SSD enables aggressive streaming. Stream music and ambient, decompress effects on load. Series S: Reduce bitrates 20% vs Series X (memory constrained).
- **PlayStation 5/4**: PS5 ultra-fast SSD perfect for streaming. PS4 HDD slower—limit concurrent streams to 2-3. Use ADPCM for effects (hardware decoder on PS4/PS5).
- **Switch**: 3GB total memory. Stream all music, decompress short effects only. Mono for most sounds. 22kHz sample rate. Aggressive compression (64-96kbps Vorbis). Test handheld mode (thermal throttling affects streaming).

## Common Pitfalls

**All Audio Set to Decompress On Load**: Developer selects all audio clips, sets Load Type = Decompress On Load for "performance". Music tracks decompress to 50MB PCM each. 10 music tracks = 500MB memory. Should use Streaming (reduces to 5-10MB buffered). Symptom: Audio memory explodes, shown in Memory Profiler. Solution: Stream music and long ambient. Decompress On Load only for short effects (<5 seconds).

**Stereo Sound Effects**: All sound effects imported as stereo. Gunshot sound = 100KB stereo, should be 50KB mono. 500 effects = 25MB wasted memory. 3D AudioSource spatialization works better with mono sources. Symptom: Memory Profiler shows stereo audio for effects. Solution: Import Settings > Force To Mono for 3D positioned sounds. Reserve stereo for music and 2D ambience.

**High Sample Rate for All Audio**: Audio imported at 48kHz (recording rate). Footsteps, UI beeps, ambient wind all at 48kHz. Wastes memory vs 22kHz (2.2x larger) with no perceptible quality benefit for non-musical sounds. Symptom: Audio memory high, quality overkill for effects. Solution: Override Sample Rate to 22050Hz for effects and ambience. Preserve 44100Hz only for music.

## Tools & Workflow

**Audio Import Settings**: Select audio clip > Inspector > Import Settings. Compression Format (Vorbis, ADPCM, MP3, PCM), Load Type (Streaming, Decompress On Load, Compressed In Memory), Sample Rate, Force To Mono, Quality slider.

**Memory Profiler**: Window > Analysis > Memory Profiler > Capture. Native Memory > Audio shows per-clip memory usage. Sort by size, identify large clips (should be streaming, not decompressed in memory).

**Audio Mixer**: Window > Audio > Audio Mixer. Create groups (Music, SFX, Ambience, UI), apply effects, manage volumes. Ducking and volume automation reduce concurrent sound complexity.

**Profiler - Audio Module**: Window > Analysis > Profiler > Audio. Shows active audio sources, CPU cost (AudioUpdate), and DSP load. Identify expensive sounds (many concurrent voices, heavy DSP effects).

**Platform-Specific Overrides**: Audio Import Settings > Platform tab (Default, PC, Xbox, PlayStation, Switch). Set per-platform compression, quality, load type. Essential for memory optimization.

**Audio Preloading Script**: Custom script to load audio via Addressables during level load. Ensures critical sounds in memory before gameplay. Use async loading (LoadAssetAsync) to avoid hitches.

**FMOD/Wwise**: Third-party audio middleware (Asset Store/external). Advanced streaming, compression, memory management vs Unity's built-in system. Used by AAA games for complex audio needs.

## Related Topics

- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction strategies
- [10.1 Asset Bundling and Streaming](10-01-AssetBundle-Fundamentals.md) - Asset loading strategies
- [6.3 Memory Bottlenecks](06-03-Memory-Bottlenecks.md) - Memory bottleneck identification
- [25.1 Build Pipeline and Workflow](25-01-Build-Optimization.md) - Build optimization

---

[← Previous: 7.4 Animation Optimization](07-04-Animation-Optimization.md) | [Next: Chapter 8 - Compression Techniques →](../chapters/08-Compression-Techniques.md)
