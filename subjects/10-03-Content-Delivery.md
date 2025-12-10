# 10.3 Content Delivery

[← Back to Chapter 10](../chapters/10-Asset-Bundling-And-Streaming.md) | [Main Index](../README.md)

Content delivery systems distribute AssetBundles via CDN hosting, downloadable content (DLC), patch management, and catalog-based versioning for post-launch updates without full rebuilds.

---

## Overview

Content delivery separates asset distribution from application distribution. Ship game with core bundles (5-10GB), host additional bundles on CDN servers (levels, DLC, seasonal content), download on-demand (players download what they need, when they need it). Benefits: smaller initial download (faster time-to-play), post-launch content updates (add levels/features without app updates), and A/B testing (serve different bundles to different player groups).

CDN (Content Delivery Network) hosting serves bundles from geographically-distributed servers. Player in Europe downloads from European server, player in Asia downloads from Asian server (lower latency, faster downloads). Popular CDNs: AWS CloudFront, Azure CDN, Google Cloud CDN, Cloudflare. Unity downloads via UnityWebRequestAssetBundle.GetAssetBundle(cdnUrl). CDN caching reduces bandwidth costs (bundles cached at edge locations, not repeatedly fetched from origin server).

DLC (Downloadable Content) delivery adds new content post-launch: expansion packs (new levels, campaigns), cosmetic items (skins, emotes), seasonal events (holiday content, limited-time modes). Implement via AssetBundles (DLC bundles hosted on CDN, downloaded when player purchases DLC). Catalog system tracks available DLC (JSON manifest listing bundle URLs, versions, dependencies). Patch management delivers bug fixes and content updates: identify changed bundles (compare bundle hashes), download only changed bundles (delta patches), replace old bundles with new versions. Version management critical (ensure players download correct versions, handle backward compatibility).

## Key Concepts

- **CDN (Content Delivery Network)**: Geographically-distributed servers caching content. Players download from nearby server (lower latency, faster speeds). Reduces origin server load (CDN serves most requests).
- **Catalog/Manifest**: JSON/XML file listing available bundles, versions, URLs, dependencies. Client downloads catalog first, compares with local bundles, identifies needed downloads. Enables version management.
- **DLC (Downloadable Content)**: Post-launch content delivered as AssetBundles. Purchased (expansion packs, cosmetic items) or free (updates, events). Downloaded on-demand (not included in initial install).
- **Delta Patching**: Downloading only changed data (binary diff between old/new bundle versions). Reduces patch sizes (download 10MB delta instead of 100MB full bundle). Requires delta generation tools.
- **Version Control**: Bundle versioning via hashes (CRC32, Hash128) or semantic versions (v1.0.0, v1.1.0). Ensures client downloads correct versions, detects corrupted downloads (hash mismatch = re-download).

## Best Practices

**CDN Setup and Integration:**
- Choose CDN provider: AWS CloudFront (global reach, good Unity integration), Azure CDN (Microsoft ecosystem), Google Cloud CDN (Google Cloud integration), Cloudflare (DDoS protection, analytics). Consider cost (per-GB bandwidth fees), latency (edge location coverage), features (HTTPS support, compression).
- Upload bundles: Build bundles locally, upload to CDN origin bucket (S3, Azure Blob Storage, Google Cloud Storage). CDN distributes to edge locations automatically (propagation takes 5-30 minutes).
- UnityWebRequest integration: Use UnityWebRequestAssetBundle.GetAssetBundle(cdnUrl, hash). Unity downloads bundle, validates hash (prevents corrupted downloads), caches locally (Caching API). Handle network errors (retry logic, fallback URLs).
- HTTPS required: Use HTTPS URLs (https://cdn.example.com/bundles/Level1.bundle). HTTP causes security warnings, app store rejections (iOS requires HTTPS). CDN provides SSL certificates (Let's Encrypt, AWS Certificate Manager).

**Catalog/Manifest System:**
- Catalog structure: JSON manifest listing bundles. Example: {"bundles": [{"name": "Level1", "url": "https://cdn.../Level1.bundle", "hash": "abc123", "size": 52428800, "dependencies": ["SharedAssets"]}]}.
- Versioning: Include catalog version number. Client downloads catalog on startup, compares with local catalog, identifies new/updated bundles. Increment version on every bundle update.
- Catalog hosting: Host catalog on CDN (highly available, low latency). Client downloads catalog before downloading bundles. Catalog small (<100KB), downloads quickly.
- Update flow: Client downloads catalog, parses JSON, compares bundle hashes with cached bundles, downloads missing/outdated bundles, updates local catalog. Background process (doesn't block gameplay).

**DLC Delivery Workflow:**
- DLC identification: Tag DLC bundles in catalog ("isDLC": true, "requiredPurchase": "expansion1"). Client checks player's purchase status (platform APIs: Steam, Xbox, PlayStation, App Store), unlocks corresponding bundles.
- Progressive download: Don't download all DLC immediately. Show DLC menu (player selects DLC to download), download on-demand. Progress bar shows download status (bytes downloaded / total bytes).
- Storage management: DLC bundles consume device storage (100MB-5GB per DLC). Implement uninstall option (delete DLC bundles, free storage). Warn player if low storage (<500MB free).
- Platform compliance: Follow platform DLC policies. Xbox/PlayStation require certification for DLC (submit bundles to platform holder). Steam/Epic allow direct CDN hosting (no certification). Mobile platforms (iOS/App Store) have <150MB initial download limit (DLC exceeding limit requires WiFi).

**Patch Management:**
- Delta patching: Generate binary diff between old/new bundle versions (tools: bsdiff, xdelta3). Distribute delta patches (10-50MB) instead of full bundles (100-500MB). Client applies patch to old bundle (reconstructs new bundle). Saves bandwidth (critical for mobile users).
- Full bundle fallback: If delta patching fails (corrupted old bundle, missing dependencies), fallback to downloading full bundle. Retry logic (attempt delta first, full download if delta fails).
- Versioning strategy: Semantic versioning (v1.0.0 = initial release, v1.1.0 = content update, v1.0.1 = hotfix). Bundle filenames include version (Level1_v1.1.0.bundle). Catalog tracks versions (client compares versions, downloads updates).
- Hotfix delivery: Critical bug fixes deployed as patch bundles. Override existing bundles (same bundle name, different hash). Client downloads on next launch, replaces old bundles. No app store approval required (bundles hosted on CDN).

**Download Size Optimization:**
- Compression: Use LZMA compression for downloadable bundles (5-10x compression). Client downloads smaller files (50MB instead of 250MB), decompresses during installation. Saves bandwidth, reduces download times.
- Asset optimization: Compress textures (BC7/ASTC), meshes (Unity mesh compression), audio (Vorbis). Reduces bundle sizes (textures = 50-70% of bundle size). See [7.1 Texture Optimization](07-01-Texture-Optimization.md).
- Bundle splitting: Split large bundles into smaller chunks (Level1_Part1, Level1_Part2). Download progressively (Part1 first, Part2 in background). Player accesses content faster (don't wait for entire 500MB bundle).
- On-demand resources: Identify rarely-used assets (cosmetics, optional content). Don't include in core bundles. Host separately, download only when player accesses feature.

**Progress Tracking and UX:**
- Download progress: Use UnityWebRequest.downloadProgress (0.0 to 1.0). Display progress bar ("Downloading Level 2: 45%"), estimated time remaining (bytes remaining / download speed).
- Cancelable downloads: Allow player to cancel downloads (UnityWebRequest.Abort()). Important for large DLC (player changes mind, low battery, cellular data limits).
- Background downloading: Download bundles while player plays other content. Queue downloads (download Level2 while playing Level1). Non-intrusive (doesn't block gameplay).
- Network error handling: Detect network errors (timeout, connection lost, 404 not found). Retry automatically (exponential backoff: retry after 1s, 2s, 4s). Show error message if retries exhausted ("Download failed. Check connection.").

**Platform-Specific:**
- **PC (Steam/Epic)**: Direct CDN hosting allowed. No download size limits. Use Steam Cloud or custom CDN. Implement auto-update (download patches on game launch).
- **Consoles (Xbox/PlayStation)**: DLC requires certification. Submit bundles to platform holder (1-2 week review process). Platform stores host DLC (player downloads from Xbox/PlayStation store). Bundles must follow platform size guidelines (Xbox: <100GB per DLC, PlayStation: <50GB).
- **Switch**: DLC hosted on Nintendo servers (requires certification). Smaller DLC preferred (<5GB) due to limited storage (32GB internal, often full). Warn player if insufficient storage.
- **Mobile (iOS/Android)**: Initial app <150MB (App Store/Google Play limit). Additional content via AssetBundles. WiFi-only downloads for >100MB bundles (cellular data warning). Use LZMA compression (smaller downloads).

## Common Pitfalls

**No Catalog Versioning**: Developer hardcodes bundle URLs in client. When bundles update, old clients download outdated bundles (version mismatch, broken content). Symptom: Players report bugs fixed in patches, old content appears. Solution: Implement catalog system. Client downloads catalog on launch, catalog lists current bundle URLs/hashes. Update catalog when bundles change (clients auto-download new bundles).

**Large Monolithic Bundles**: Developer packages entire level in single 500MB bundle. Player must download 500MB before playing level (5-10 minute wait on slow connections). Symptom: Player complaints about long download times, abandonment during download. Solution: Split bundles (Level1_Environment: 200MB, Level1_Characters: 100MB, Level1_Effects: 50MB). Download Environment first (player sees level), Characters/Effects load in background (progressive experience).

**No Retry Logic**: Developer downloads bundle via UnityWebRequest, assumes success. Network interruptions (WiFi dropout, mobile signal loss) cause download failures (player gets error, can't play content). Symptom: "Download failed" errors, angry player reviews. Solution: Implement retry logic with exponential backoff. Retry failed downloads automatically (3-5 attempts). Resume downloads (use HTTP Range header to continue from last byte).

**No WiFi Warning on Mobile**: Developer downloads 200MB DLC on mobile via cellular data. Player exceeds data cap (expensive overage charges), blames game. App store reviews complain about excessive data usage. Symptom: 1-star reviews "game used all my data". Solution: Detect network type (WiFi vs cellular). Prompt player before large downloads on cellular ("This download is 200MB. Continue on cellular or wait for WiFi?"). Default to WiFi-only for >100MB downloads.

## Tools & Workflow

**UnityWebRequestAssetBundle**: Unity API for downloading bundles. UnityWebRequestAssetBundle.GetAssetBundle(url, hash). Features: automatic caching (Caching API), hash validation (prevents corrupted downloads), progress tracking (downloadProgress property). Use for CDN-hosted bundles.

**Addressables Remote Hosting**: Addressables > Hosting > Remote Catalog. Builds catalog automatically, hosts bundles locally (testing) or uploads to CDN. Simplifies content delivery (no manual catalog management). Recommended for new projects.

**CDN Providers**: AWS CloudFront (https://aws.amazon.com/cloudfront/), Azure CDN (https://azure.microsoft.com/en-us/services/cdn/), Google Cloud CDN (https://cloud.google.com/cdn), Cloudflare (https://www.cloudflare.com/cdn/). Upload bundles to origin storage (S3, Azure Blob, GCS), CDN distributes to edge locations.

**Delta Patching Tools**: bsdiff/bspatch (http://www.daemonology.net/bsdiff/) generates binary diffs. xdelta3 (http://xdelta.org/) alternative tool. Integrate into build pipeline (compare old/new bundles, generate delta patches, upload to CDN).

**Catalog Management**: Custom tools or Unity Addressables. Catalog builder script: scans built bundles, generates JSON manifest (bundle names, URLs, hashes, sizes, dependencies). Upload catalog to CDN, client downloads on launch.

**Download Manager**: Custom AssetBundleDownloadManager. Queues downloads (download one bundle at a time or parallel downloads), tracks progress (aggregate progress across multiple bundles), handles retries (exponential backoff), validates hashes (re-download if mismatch).

**Platform APIs**: Steam (Steamworks API, Steam Cloud), Xbox (XDK/GDK package delivery), PlayStation (DevNet SDK DLC APIs), Switch (Nintendo Developer Portal), iOS/Android (in-app purchase verification). Integrate to check DLC ownership, unlock content.

## Related Topics

- [10.1 AssetBundle Fundamentals](10-01-AssetBundle-Fundamentals.md) - Bundle creation and compression
- [10.2 Loading Strategies](10-02-Loading-Strategies.md) - Downloading and loading bundles
- [10.4 Platform-Specific Considerations](10-04-Platform-Specific-Considerations.md) - Platform delivery requirements
- [8.4 Data Compression](08-04-Data-Compression.md) - LZMA compression for downloads

---

[← Previous: 10.2 Loading Strategies](10-02-Loading-Strategies.md) | [Next: 10.4 Platform-Specific Considerations →](10-04-Platform-Specific-Considerations.md)
