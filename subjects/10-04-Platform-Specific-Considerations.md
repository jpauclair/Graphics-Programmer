# 10.4 Platform-Specific Considerations

[← Back to Chapter 10](../chapters/10-Asset-Bundling-And-Streaming.md) | [Main Index](../README.md)

Platform-specific AssetBundle constraints include storage limits, certification requirements, DLC policies, compression formats, and delivery mechanisms unique to PC, consoles, and mobile platforms.

---

## Overview

Each platform enforces unique AssetBundle constraints. PC platforms (Steam, Epic) allow direct CDN hosting with minimal restrictions (unlimited bundle sizes, no certification). Console platforms (Xbox, PlayStation, Switch) require certification for DLC bundles (1-2 week review process), enforce download size limits (Xbox: <100GB per DLC, PlayStation: <50GB), and mandate platform-specific compression formats. Mobile platforms (iOS, Android) limit initial app size (<150MB), require WiFi warnings for large downloads (>100MB), and have strict app store policies (no executable code in bundles).

Storage considerations vary drastically: PC users have 500GB-2TB drives (large bundles acceptable), console users have 500GB-1TB (manageable but limited), Switch users have 32GB internal storage (often <10GB free, small bundles critical), mobile users have 32-256GB (shared with photos/apps, storage-conscious design mandatory). Certification requirements: PC requires none (instant updates), consoles require platform holder approval (Xbox/PlayStation certification process), mobile requires app store review (1-7 days).

Compression and format requirements: PC/Xbox use BC texture compression (BC7, BC5, BC6H), PlayStation uses BC or proprietary formats (GNM-optimized), Switch mandates ASTC compression (6x6 to 12x12 blocks), mobile uses ASTC (Android) or PVRTC/ASTC (iOS). Bundle building must target correct platform (Unity BuildTarget.StandaloneWindows64, BuildTarget.XboxOne, BuildTarget.PS5, BuildTarget.Switch, BuildTarget.iOS).

## Key Concepts

- **Certification Requirements**: Console DLC bundles require platform holder approval (submit to Xbox/PlayStation/Nintendo for testing). Review process takes 1-2 weeks. Bundles must meet technical requirements (no crashes, proper entitlements, size limits).
- **Storage Limits**: Platforms impose storage constraints. Switch has 32GB internal storage (often <10GB free), mobile devices 32-256GB (shared with user content). Bundles must fit within available storage or provide uninstall options.
- **Platform BuildTargets**: Unity builds platform-specific bundles. PC uses BC compression, Switch uses ASTC, mobile uses ASTC/PVRTC. Build separate bundles per platform (same logical content, different formats).
- **DLC Entitlements**: Console platforms verify DLC ownership via entitlement checks (Xbox: XStore, PlayStation: PSN, Switch: Nintendo eShop). Bundles locked until player purchases DLC. PC uses Steam API/Epic SDK for ownership checks.
- **Executable Code Restrictions**: Mobile app stores prohibit executable code in bundles (no .dll, .so files). Bundles contain only data (assets, textures, meshes, audio). Violating policy causes app rejection.

## Best Practices

**PC Platform (Steam/Epic/GOG):**
- Direct CDN hosting: Host bundles on custom CDN (AWS, Azure, Cloudflare). No platform restrictions. Update bundles anytime (no certification).
- Large bundles acceptable: PC users have 500GB-2TB storage. Bundles can be 100-500MB without issues. Use BC7 compression for textures (best quality/size ratio).
- Auto-update integration: Steam Workshop or custom update system. Check for bundle updates on game launch, download automatically (background process).
- Mod support: AssetBundles enable modding. Players create custom bundles (new levels, characters, skins), load via modding API. Popular on Steam (Workshop integration).
- Quality tiers: PC hardware varies widely. Provide multiple bundle variants (TexturesLow.bundle for GTX 1060, TexturesHigh.bundle for RTX 4090). Select based on detected GPU.

**Xbox Platform:**
- Smart Delivery: Xbox Series X/S and Xbox One use same game package. AssetBundles can provide platform-specific assets (Series X = 4K textures, Xbox One = 1080p textures). Load appropriate bundles at runtime.
- DLC certification: Submit DLC bundles to Microsoft for certification. Process takes 1-2 weeks. Must pass WACK (Windows App Certification Kit) tests, meet size limits (<100GB per DLC).
- XStore integration: Use Xbox Services API to check DLC ownership. Player purchases DLC on Xbox Store, game queries entitlements, unlocks corresponding bundles.
- Title storage: Use Xbox Live Title Storage (cloud storage for bundles). Players download bundles from Microsoft servers (no custom CDN needed). 10GB free storage per title.
- GDK build: Use GDK (Game Development Kit) with BuildTarget.GameCoreXboxSeries for Series X/S, BuildTarget.GameCoreXboxOne for Xbox One. Use BC7/BC5 compression (DirectX 12 formats).

**PlayStation Platform:**
- Package structure: PlayStation uses .pkg files for DLC. Build AssetBundles, package into .pkg via PlayStation SDK tools (orbis-pub-cmd for PS4, prospero-pub-cmd for PS5).
- DLC certification: Submit DLC packages to Sony for certification. 1-2 week review process. Must meet Technical Requirements Checklist (TRC), size limits (<50GB per DLC).
- PSN entitlements: Use PlayStation SDK APIs (sceNpCheckEntitlement) to verify DLC ownership. Query PSN for player's purchase history, unlock bundles.
- UMA optimization: PlayStation uses unified memory architecture (16GB shared). Bundle streaming efficient (fast SSD, no separate GPU memory transfers). Use LZ4 compression (fast decompression).
- GNM/GNMX build: Use BuildTarget.PS4 or BuildTarget.PS5. Textures use BC7/BC5 or GNM-optimized formats (provided by SDK). Shaders compile to PlayStation-specific ISA.

**Nintendo Switch Platform:**
- ASTC mandatory: Switch GPU supports ASTC compression only (no BC formats). Build bundles with BuildTarget.Switch, Unity converts textures to ASTC automatically (6x6, 8x8, 12x12 blocks).
- Small bundles critical: Switch has 32GB internal storage (often <10GB free after OS, games, save data). Bundles must be <50MB ideally, <200MB max. Larger bundles risk "insufficient storage" errors.
- Docked vs handheld: Switch renders at 1080p docked, 720p handheld. Provide resolution-appropriate bundles (1080p textures docked, 720p handheld). Detect mode at runtime (Application.isMobilePlatform).
- DLC certification: Submit DLC to Nintendo for approval (Lotcheck process). 1-2 week review. DLC hosted on Nintendo servers (players download from eShop).
- eShop integration: Use nn::ec (e-commerce SDK) to check DLC ownership. Query eShop for player's purchases, unlock bundles.

**Mobile Platforms (iOS/Android):**
- Initial size limit: App stores limit initial download. iOS App Store: <150MB (larger requires WiFi warning), Google Play: <150MB for APK (additional assets via Expansion Files or PAD). AssetBundles deliver post-install content.
- WiFi warnings: For downloads >100MB, prompt player ("Large download. Continue on cellular or wait for WiFi?"). iOS requirement (App Store rejection if not implemented). Android recommended (user data limits).
- ASTC compression: Use ASTC for Android (Adreno, Mali, PowerVR GPUs support ASTC). iOS supports ASTC or PVRTC (older devices). Unity handles format selection automatically.
- No executable code: App stores prohibit executable code in bundles (.dll, .so). Include only data (textures, meshes, audio, materials). Managed code acceptable (C# .NET assemblies precompiled in app binary).
- Storage management: Mobile storage limited (32-256GB, shared with photos/apps). Implement uninstall option (delete DLC bundles). Warn if <500MB free storage.
- In-app purchases: Use platform APIs (iOS StoreKit, Android Google Play Billing) to verify DLC purchases. Unlock bundles after purchase verification.

**Cross-Platform Bundles:**
- Separate builds per platform: Build bundles for each platform (PC, Xbox, PlayStation, Switch, iOS, Android). Same logical content, different formats (BC7 vs ASTC vs PVRTC).
- Catalog-based selection: Catalog includes platform field ({"platform": "PC", "url": "...PC.bundle"}, {"platform": "Switch", "url": "...Switch.bundle"}). Client detects platform (Application.platform), loads appropriate bundles.
- Build automation: Automate multi-platform builds (Jenkins, GitHub Actions). Script builds bundles for all platforms, uploads to CDN (PC/Xbox/PS to AWS, Switch/mobile to respective stores).
- Testing: Test bundles on all target platforms (PC, Xbox dev kit, PlayStation dev kit, Switch dev kit, iOS device, Android device). Verify formats load correctly, no missing assets.

## Common Pitfalls

**Using PC Bundles on Switch**: Developer builds bundles with BuildTarget.StandaloneWindows64, deploys to Switch. Textures use BC7 compression (not supported on Switch). Game crashes or shows black textures. Symptom: "Unsupported texture format" errors, black/missing textures on Switch. Solution: Build Switch-specific bundles (BuildTarget.Switch). Unity converts textures to ASTC automatically.

**No WiFi Warning on Mobile**: Developer downloads 200MB DLC on mobile without checking network type. Player on cellular downloads 200MB (exceeds data cap, expensive overage charges). App store reviews complain about excessive data usage. iOS app rejected for not prompting WiFi. Symptom: 1-star reviews "game used all my data", app rejection. Solution: Detect network type (Application.internetReachability == NetworkReachability.ReachableViaCarrierDataNetwork). Prompt before large downloads on cellular.

**Exceeding Switch Storage**: Developer packages 500MB DLC bundle for Switch. Switch users have <10GB free storage (32GB internal often full). Download fails ("insufficient storage"). Players unable to install DLC. Symptom: Players report "cannot download DLC", "not enough space" errors. Solution: Split into smaller bundles (<50MB each), implement progressive download (download Part1, then Part2 as needed). Warn player of storage requirements before download.

**No DLC Entitlement Check**: Developer hosts DLC bundles on public CDN, no ownership verification. Players share DLC URLs (bypass purchase, download DLC for free). Lost revenue, platform policy violation. Symptom: Players sharing DLC links on forums, unexpected DLC usage spikes. Solution: Implement entitlement checks (Xbox: XStore API, PlayStation: PSN API, Steam: Steamworks API). Verify purchase before providing bundle URLs. Use signed/encrypted URLs (expire after 1 hour).

## Tools & Workflow

**Unity BuildPipeline**: BuildPipeline.BuildAssetBundles(outputPath, BuildAssetBundleOptions, BuildTarget). BuildTarget specifies platform (StandaloneWindows64, XboxOne, PS5, Switch, iOS, Android). Unity handles platform-specific conversions (texture compression, shader compilation).

**Platform SDKs**: Xbox GDK (Game Development Kit), PlayStation SDK (Orbis SDK for PS4, Prospero SDK for PS5), Nintendo Switch SDK (NintendoSDK), iOS Xcode, Android SDK. Required for building and testing platform-specific bundles.

**Addressables Platform Support**: Addressables automatically builds platform-specific bundles. Configure Addressables Groups per platform (PC Group, Console Group, Mobile Group). Addressables selects appropriate bundles at runtime (detects Application.platform).

**Platform Profilers**: Xbox PIX (GPU profiling), PlayStation Razor/Tuner (CPU/GPU profiling), Switch NNGFX/LLGD (GPU profiling). Profile bundle loading times, identify slow bundles (LZMA decompression, large textures), optimize accordingly.

**Certification Tools**: Microsoft WACK (Windows App Certification Kit) for Xbox, Sony TRC Checker for PlayStation, Nintendo Lotcheck for Switch. Validate bundles meet platform requirements before submission (avoid certification rejections).

**Storage Analysis**: Platform-specific storage APIs (Xbox: XDK storage APIs, PlayStation: SceAppContent, Switch: nn::fs). Query available storage before downloading bundles, warn player if insufficient space.

**Entitlement APIs**: Xbox: XStore API (XStoreQueryEntitlementsAsync), PlayStation: sceNpCheckEntitlement, Steam: ISteamApps::BIsDlcInstalled, Epic: EOS_Ecom_QueryOwnership. Verify DLC ownership before unlocking bundles.

## Related Topics

- [10.1 AssetBundle Fundamentals](10-01-AssetBundle-Fundamentals.md) - Bundle creation and structure
- [10.3 Content Delivery](10-03-Content-Delivery.md) - DLC and patching
- [9.1 PC Platform Guidelines](09-01-PC-Platform.md) - PC-specific asset formats
- [9.2 Xbox Platform Guidelines](09-02-Xbox-Platform.md) - Xbox-specific considerations
- [9.3 PlayStation Platform Guidelines](09-03-PlayStation-Platform.md) - PlayStation-specific considerations
- [9.4 Nintendo Switch Platform Guidelines](09-04-Nintendo-Switch-Platform.md) - Switch-specific constraints

---

[← Previous: 10.3 Content Delivery](10-03-Content-Delivery.md) | [Next: Chapter 11 →](../chapters/11-Materials-And-Textures.md)
