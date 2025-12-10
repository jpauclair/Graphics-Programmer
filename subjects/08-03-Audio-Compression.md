# 8.3 Audio Compression

[← Back to Chapter 8](../chapters/08-Compression-Techniques.md) | [Main Index](../README.md)

Audio compression reduces file sizes and memory by 80-95% using lossy formats (Vorbis, MP3, ADPCM) while maintaining acceptable quality for game audio.

---

## Overview

Uncompressed audio is massive: 1 minute stereo at 44.1kHz = 10.3MB PCM (44,100 samples/sec × 2 channels × 2 bytes = 176KB/sec × 60 sec). A game with 30 music tracks (3 minutes average) = 930MB uncompressed. With 500 sound effects (2 seconds average) = 1.7GB total uncompressed audio. Compression via Vorbis (10:1 ratio at 128kbps) reduces this to 170MB—90% reduction.

Audio compression uses perceptual coding: removes frequencies humans cannot hear (psychoacoustic masking). Vorbis and MP3 achieve 10:1 compression with near-transparent quality. ADPCM provides 4:1 compression with fast hardware decompression (ideal for short effects). Format choice depends on audio type: Vorbis for music (best quality/size), ADPCM for effects (fast decompression), and PCM only for tiny UI sounds (<0.5 seconds, frequently played).

Optimization workflow: audit audio assets (identify uncompressed), select format per audio type (music = Vorbis streaming, effects = ADPCM decompress on load), set bitrates (128kbps music, 64-96kbps effects), and validate quality (A/B test compressed vs source). Platform considerations: mobile prefers MP3 (hardware decoders), consoles use Vorbis or platform-specific (XMA on Xbox, ATRAC9 on PlayStation).

## Key Concepts

- **Vorbis**: Open-source lossy compression. 10:1 ratio at 128kbps, 15:1 at 96kbps. Excellent quality. Variable bitrate (VBR) adapts to content complexity. Default choice for music and ambient.
- **MP3**: Ubiquitous lossy format. 10:1 ratio at 128kbps. Hardware decoders on mobile (better battery). Slightly lower quality than Vorbis at equivalent bitrates. Good for mobile platforms.
- **ADPCM**: Adaptive Delta PCM. 4:1 compression ratio. Fast decompression (hardware accelerated on consoles). Lower quality than Vorbis/MP3. Ideal for short effects (<5 seconds).
- **Bitrate**: Compressed audio data rate (kbps = kilobits per second). Higher = better quality, larger files. 64kbps = low quality (voices acceptable, music poor). 128kbps = good quality (music acceptable). 192kbps = high quality (near CD).
- **Sample Rate**: Audio frequency (samples per second). 44.1kHz = CD quality. 22kHz = half rate (sufficient for effects). 48kHz = professional (overkill for games). Lower sample rate halves file size.

## Best Practices

**Vorbis Compression:**
- Use Vorbis for music and long ambient: 128kbps provides excellent quality (transparent for most listeners). Import Settings > Compression Format = Vorbis, Quality = 50-70 (128kbps).
- Variable bitrate (VBR): Vorbis VBR adapts bitrate to content complexity. Simple passages (silence, drones) use fewer bits, complex sections (orchestral, dense mix) use more. Improves quality without increasing average bitrate.
- Streaming mode: Music and long ambient should stream (Load Type = Streaming). Loads in chunks, reducing memory 10-50x. No noticeable loading cost (streams from disk/SSD).
- Bitrate recommendations: Background music (96kbps), main theme (128kbps), high-fidelity music (160-192kbps). Higher bitrates waste space—128kbps is sweet spot.

**MP3 Compression:**
- Use MP3 on mobile platforms: iOS and Android have hardware MP3 decoders (better battery life vs software Vorbis). Import Settings > Platform Override > iOS/Android > MP3.
- Bitrate selection: 96-128kbps for mobile music. 64kbps for mobile effects (acceptable quality on small speakers). Test on target devices (earbuds vs speaker output).
- Constant bitrate (CBR) vs variable (VBR): CBR easier for streaming (predictable data rate). VBR better quality for same average bitrate. Use VBR for music, CBR for streaming ambient.
- Encoding quality: Unity uses LAME encoder (high quality). Quality slider in import settings controls encoding parameters (higher = slower encode, better quality).

**ADPCM Compression:**
- Use ADPCM for short sound effects: 4:1 compression, fast decompression. Ideal for <5 second sounds (footsteps, gunshots, UI). Import Settings > Compression Format = ADPCM.
- Hardware decompression on consoles: Xbox and PlayStation have dedicated ADPCM hardware decoders. Decompression essentially free (CPU cost negligible). Enable on console builds.
- Decompress On Load: ADPCM effects should use Load Type = Decompress On Load. Decompresses at load time, stores PCM in memory. Fast playback without runtime decompression cost.
- Quality is acceptable: ADPCM has audible artifacts (quantization noise) but acceptable for short effects. Avoid for music or voice (use Vorbis/MP3 instead).

**Sample Rate Optimization:**
- Use 44.1kHz for music: Standard CD quality. Sufficient for all music. Import Settings > Sample Rate Setting = Preserve Sample Rate (maintains source rate).
- Use 22kHz for non-musical sounds: Footsteps, ambient, UI, effects. Halves file size vs 44.1kHz. Quality loss imperceptible for non-tonal content. Sample Rate Setting = Override Sample Rate > 22050Hz.
- Use 48kHz only for professional audio: Cinema/broadcast standard. Overkill for games. Avoid (8% larger files vs 44.1kHz for no benefit).
- Match source rate when possible: If source is 22kHz, don't upsample to 44.1kHz (wastes space). If source is 48kHz, downsample to 44.1kHz (saves space, no audible difference).

**Load Type and Memory Management:**
- Stream music and long ambient: Load Type = Streaming. Music tracks (2-5 minutes) consume 20-50MB PCM, but only 2-5MB buffered when streaming. Essential for memory management.
- Decompress On Load for short effects: Load Type = Decompress On Load. Effects <5 seconds decompress at load, store PCM in memory. Fast playback (no runtime cost), acceptable memory (<1MB per effect).
- Compressed In Memory for rare sounds: Load Type = Compressed In Memory. Stores compressed, decompresses during playback (CPU cost). Good for infrequent sounds (death screams, special abilities). Smallest memory, slight CPU cost.
- Preload critical sounds: Load essential sounds (gunfire, player footsteps, UI beeps) during level load. Avoid runtime loading (causes hitches).

**Platform-Specific:**
- **PC**: Vorbis for all audio. 128kbps for music, 64-96kbps for effects. Decompress On Load for effects, Streaming for music. PC CPUs handle Vorbis decompression easily.
- **Xbox Series X/S**: Vorbis or XMA (Xbox proprietary). XMA slightly better quality at same bitrate. Streaming works well (fast SSD). ADPCM for effects (hardware decoder).
- **PlayStation 5/4**: Vorbis or ATRAC9 (PlayStation proprietary). ATRAC9 optimized for PS hardware. Streaming excellent on PS5 (ultra-fast SSD). ADPCM for effects.
- **Switch**: Vorbis at lower bitrates (64-96kbps). Memory constrained (3GB total). Stream all music, decompress short effects only. 22kHz sample rate for effects.
- **Mobile**: MP3 for iOS/Android (hardware decoders). 96kbps music, 64kbps effects. Streaming from flash storage (fast). Battery life critical (hardware decode saves power).

## Common Pitfalls

**All Audio Decompress On Load**: Developer sets Load Type = Decompress On Load for all audio (music, effects, ambient). Music tracks decompress to 50MB PCM each. 10 music tracks = 500MB memory. Should stream music (reduces to 5-10MB buffered). Symptom: Audio memory dominates Memory Profiler. Solution: Stream music and long ambient (>10 seconds). Decompress On Load only for short effects.

**High Sample Rate for All Audio**: Audio imported at 48kHz (recording rate). Footsteps, UI beeps, ambient birds all at 48kHz. Wastes 8% memory vs 44.1kHz, 220% vs 22kHz for effects. No perceptible quality benefit for non-musical sounds. Symptom: Audio files larger than necessary. Solution: Override Sample Rate to 22050Hz for effects and ambient. Preserve 44100Hz only for music.

**Uncompressed Audio (PCM)**: Audio imported without compression. 1-minute music track = 10.3MB uncompressed. Should be ~1MB Vorbis 128kbps (90% reduction). Symptom: Huge audio memory usage, large build sizes. Solution: Enable compression (Vorbis or ADPCM) on all audio except tiny UI sounds (<0.5 seconds, frequently played where PCM avoids decompression CPU cost).

## Tools & Workflow

**Audio Import Settings**: Select audio clip > Inspector > Import Settings. Compression Format (PCM, Vorbis, ADPCM, MP3), Load Type (Streaming, Decompress On Load, Compressed In Memory), Sample Rate Setting, Quality slider.

**Platform-Specific Overrides**: Audio Import Settings > Platform tab. Override compression, bitrate, sample rate per platform (PC, Xbox, PlayStation, Switch, iOS, Android). Essential for memory optimization.

**Memory Profiler**: Native Memory > Audio shows per-clip memory usage. Streaming clips show small buffer size (5-10MB). Decompress On Load shows full PCM size. Identify memory hogs.

**Audacity**: Free audio editor for testing compression. Export with different formats/bitrates, A/B compare quality. Validates bitrate selection before importing to Unity. Download from https://www.audacityteam.org/

**Audio Quality Comparison Script**: Custom Unity script playing source audio vs compressed side-by-side. Toggle between versions, validate quality acceptable. Essential for finding optimal bitrate (balance quality vs size).

**Profiler - Audio Module**: Window > Analysis > Profiler > Audio. Shows active audio sources, CPU cost (AudioUpdate, decompression), and memory. Identify expensive sounds (heavy decompression, many concurrent voices).

**FMOD/Wwise**: Third-party audio middleware (paid). Advanced compression, streaming, memory management vs Unity's built-in system. Used by AAA games. Supports platform-specific codecs (XMA, ATRAC9) automatically.

## Related Topics

- [7.5 Audio Asset Optimization](07-05-Audio-Asset-Optimization.md) - Audio optimization strategies
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction
- [10.1 Asset Bundling and Streaming](10-01-Asset-Bundling.md) - Audio streaming via AssetBundles
- [6.3 Memory Bottlenecks](06-03-Memory-Bottlenecks.md) - Memory bottleneck identification

---

[← Previous: 8.2 Mesh Compression](08-02-Mesh-Compression.md) | [Next: 8.4 Data Compression →](08-04-Data-Compression.md)
