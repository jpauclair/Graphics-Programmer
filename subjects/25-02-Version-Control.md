# 25.2 Version Control for Graphics Development

[← Back to Chapter 25](../chapters/25-Build-Pipeline-And-Workflow.md) | [Main Index](../README.md)

Version control essential for team collaboration (merge conflicts in art assets = painful). Graphics-specific challenges: binary assets (textures, meshes, shaders = don't diff meaningfully), large files (10-100 GB project = network slow), frequent changes (artists update textures daily = merge conflicts). Strategies: (1) separate art/code (code in Git, art in Perforce/Plastic), (2) branching (feature branches = artist A works on character, artist B on environment = no conflicts), (3) lock-based (asset locked during edit = no concurrent edits = simpler).

---

## Overview

Graphics Workflow: artists create/modify textures (Photoshop), models (Blender, Maya), materials (Unity editor). Commits = new revisions. Main branch = latest playable. Branching = feature work (separate branch, merge back when ready). Merging = conflicts if two artists edit same asset = complex (binary merge tools limited). Lock-based system = asset locked during edit (only one artist at time = no conflicts, but waiting). Best: team workflow planned (avoid simultaneous edits), or use specialized VCS (Perforce, Plastic = binary-friendly).

Repository Size: typical game 50-200 GB (depends on asset quality). Cloud storage (GitHub 100 GB limit, GitLab unlimited = preferred). On-disk storage = SSD critical (HDD = network access slow, defeats purpose).

## Key Concepts

- **Git for Graphics**: distributed VCS, text-friendly (code), binary-hostile (assets don't compress well, history bloats). Challenge: 1GB texture committed = 1GB history forever (if deleted later). Solution: Git LFS (Large File Storage, commits to external storage, references in repo = smaller repo size). Used by: indie teams, code-heavy projects.
- **Perforce**: centralized VCS, binary-optimized (store deltas = small repository), lock-based (asset checked out, locked, modified, checked in = no conflicts). Cost: enterprise license. Benefit: designed for big teams, large assets, complex workflows. Used by: AAA studios (industry standard).
- **Plastic SCM**: centralized like Perforce, but cloud-native (cloud servers available), branch-friendly (full branch history), free for small teams. Benefit: middle ground (Perforce complexity, Git ease). Used by: studios trying Git alternative.
- **Branching Strategy**: main = always playable, develop = daily work, feature branches = artist-specific (25-Ambient-Design = one branch, 25-Character-Redesign = another). Merge to develop when complete (code review equivalent for art = lead artist approval). Merge to main for release.
- **Asset Lock Systems**: lock file (asset.psd.lock indicates locked), prevent concurrent edits. When artist finishes, lock released = next artist can edit. Benefit: zero merge conflicts (one editor at time). Risk: bottleneck (artist waiting for lock release = slow iteration).

## Best Practices

**Repository Organization**:
- Separate Art/Code: Graphics assets live in managed folders (Artists folder = shared location), code in separate (Code folder). Different versioning (code = Git, art = Perforce if needed).
- .gitignore for graphics: exclude Editor caches (Library/ = compiled cache = 1-2 GB, don't commit). Commit only source assets (textures PSD, not DDS compiled version).
- Lock-based for team: if small team (<5 artists), simpler workflow = one artist per asset.

**Merge Conflict Avoidance**:
- Task division: artist A handles characters, artist B handles environment = no simultaneous asset edits.
- Code-centric teams: if primarily code, smaller asset branches = easier merges.
- Regular syncs: pull latest daily = reduce conflict scope.

**Large Asset Handling**:
- Git LFS: `git lfs track "*.psd" "*.unitypackage"` = LFS manages large files, repository stays small.
- Perforce: native support (configured per file type = .psd, .fbx = large file handlers).
- Cloud storage: if teams distributed globally, cloud VCS (GitHub, Plastic Cloud) = faster access than local server.

**Platform-Specific**:
- **Indie/Small Teams**: Git + LFS (free, familiar), or Plastic (free tier available).
- **AAA/Enterprise**: Perforce (industry standard, binary optimized), or Plastic (easier onboarding than Perforce).
- **Mobile-First**: smaller assets = Git fine, LFS not needed.

## Common Pitfalls

**Repository Bloat**: Developer commits large raw textures (8K PSD = 500MB), then deletes. Repository = 500MB committed forever (history keeps it). Symptom: clone slow (50 GB repository), pull slow. Solution: use LFS from start (track large file types before committing), or rewrite history (BFG Repo-Cleaner, dangerous = loses history).

**Merge Conflicts Everywhere**: Small team, all artists editing environment material simultaneously. Merge conflicts = complex (binary FBX files can't auto-merge). Symptom: merge fails, manual resolution needed. Solution: divide work (artist per area), or lock-based system (Perforce = one editor at time).

**Network Bottleneck**: Project on NAS (network attached storage), each pull = network access = slow (5-30 MB/sec = 1 hour to pull 50 GB). Symptom: team frustrated (pulls slow). Solution: local storage (SSD clone on each machine), or cloud VCS (GitHub, Plastic = globally distributed servers = faster).

## Tools & Workflow

**Git LFS Setup**:
```bash
git lfs install
git lfs track "*.psd" "*.fbx" "*.unitypackage"
git add .gitattributes
git commit -m "Setup LFS"
```

**Gitignore for Graphics**:
```
/Library/
/Temp/
/Build/
*.apk
*.ipa
.DS_Store
Thumbsdb
```

**Perforce Client Workflow** (simplified):
```bash
p4 clients # List workspaces
p4 sync    # Pull latest
p4 edit asset.psd # Lock asset for editing
# Edit in Photoshop
p4 submit -d "Updated environment material" # Commit and unlock
```

**Branch Protection**: setup rules (main branch = PR required, code review before merge = quality gate).

## Related Topics

- [25.1 Build Optimization](25-01-Build-Optimization.md) - Build pipeline
- [25.3 Continuous Integration](25-03-Continuous-Integration.md) - Automated testing
- [25.4 Asset Pipeline](25-04-Asset-Pipeline.md) - Asset import workflow
- [04.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) - Team tools

---

[Previous: 25.1 Build Optimization](25-01-Build-Optimization.md) | [Next: 25.3 Continuous Integration →](25-03-Continuous-Integration.md)
