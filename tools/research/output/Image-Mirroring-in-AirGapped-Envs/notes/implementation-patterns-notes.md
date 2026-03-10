# Image Mirroring in AirGapped Environments — Implementation Patterns Notes

<!-- Notes for the Implementation Patterns subtopic of Image Mirroring in AirGapped Environments. Organized by question, not source. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->
<!-- Citations are stored in whitepaper/Image-Mirroring-in-AirGapped-Envs-references.md -->

## Questions

### Q1: What are the architectural patterns for implementing image mirroring in air-gapped environments?

**Pattern 1: Partially Disconnected (Bastion Host)**
- A bastion or jump host has controlled internet access
- Mirror tool runs on bastion, pulls from public registries
- Pushes directly to internal mirror registry
- Suitable for environments with limited but available connectivity [8]

**Pattern 2: Fully Disconnected (Sneakernet)**
- Connected system mirrors images to portable media (tarball/disk)
- Physical transfer of media across the air gap
- Disconnected system loads images from media to local registry
- Required for true air-gapped environments with no network path [8][3]

**Pattern 3: Hauler's Unified Approach**
- Single binary handles fetch, store, package, and distribute operations
- Stores content as OCI Compliant Artifacts
- Provides embedded registry (port 5000) and embedded fileserver (port 8080) for bootstrapping [3]
- Workflow:
  1. `hauler store add` - fetch artifacts on connected system
  2. `hauler store save` - package to compressed archive (haul.tar.zst)
  3. Transfer archive across air gap
  4. `hauler store load` - import on disconnected system
  5. `hauler store serve registry` - serve as local registry [3]

**Pattern 4: Red Hat oc-mirror Workflow**
- Declarative image set configuration (YAML) specifies what to mirror [8]
- Supports both partially and fully disconnected environments [8]
- Maintains metadata for incremental updates [8]
- Generates supporting artifacts for cluster configuration [8]
- Steps for fully disconnected:
  1. Run `oc mirror --config=./imageset-config.yaml file://<path>` to create tarball
  2. Transfer tarball to disconnected environment
  3. Run `oc mirror` from disk to mirror registry [8]

**Pattern 5: MicroShift/OpenShift Mirror Registry**
- Get container image list to be mirrored
- Configure mirroring prerequisites (pull secrets, credentials)
- Download images on internet-connected host
- Copy downloaded image directory to air-gapped site
- Upload images to mirror registry
- Configure hosts to use mirror registry [6]

### Q2: How do organizations handle image synchronization and updates in air-gapped environments?

**Initial Sync vs. Incremental Updates:**
- Initial image set download is typically the largest transfer [8]
- Subsequent runs only download changed images since last execution [8]
- oc-mirror references metadata from storage backend to determine deltas [8]
- Metadata must be preserved between runs—do not delete or modify [8]

**Update Workflow:**
- Run mirroring tool with same configuration as initial sync [8]
- Tool compares current state with metadata to identify new/changed images
- Only new content is downloaded and transferred
- Provides update paths for platform and operators with dependency resolution [8]

**Image Pruning:**
- oc-mirror can prune images from target registry that were excluded from configuration since previous execution [8]
- Helps manage storage in mirror registries

**Considerations:**
- Storage backend can be local directory or docker v2 registry [8]
- Must use same storage backend for all runs targeting same mirror registry [8]
- Depending on configuration, mirroring can download hundreds of gigabytes [8]

**Hauler's Approach:**
- Stores artifacts with metadata (attestations, SBOMs, signatures) [3]
- `hauler store info` shows all content with type, platform, layers, and size [3]
- Supports multi-architecture images (linux/amd64, linux/arm64) [3]

---

## Summary

Implementation patterns for air-gapped image mirroring fall into two main categories: partially disconnected (with controlled internet access via bastion hosts) and fully disconnected (requiring physical media transfer). Tools like Hauler and oc-mirror provide structured workflows for both scenarios, with Hauler offering a lightweight single-binary approach with embedded registry capabilities, while oc-mirror provides enterprise features like declarative configuration and incremental updates [3][8].

Organizations handle synchronization through incremental mirroring—after the initial large transfer, subsequent updates only include changed images. This requires preserving metadata between runs and using consistent storage backends. The key architectural decision is whether to use direct registry-to-registry mirroring (when partial connectivity exists) or disk-based transfer (for fully air-gapped environments) [8].
