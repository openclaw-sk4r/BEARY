# Image Mirroring in AirGapped Environments — Notes

<!-- Notes for the Image Mirroring in AirGapped Environments topic. Organized by question, not source. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->
<!-- Citations are stored in whitepaper/Image-Mirroring-in-AirGapped-Envs-references.md -->

## General Understanding

### Q1: What is container image mirroring and how does it work?

Container image mirroring is the process of copying container images from one registry (typically a public registry like Docker Hub or quay.io) to another registry (typically a private or local registry). This creates a local copy of the images that can be accessed without connecting to the original source [1].

**How it works:**
- A container image is a self-contained, executable bundle containing everything required to run software in a well-defined runtime environment [2]
- Images consist of one or more tar archives plus a JSON manifest file describing the software and how to run it [2]
- The manifest contains references to the config layer and filesystem layers (tarballs) that make up the container [2]
- Tools like skopeo and crane interact with registries via well-defined APIs (OCI Distribution Spec) without requiring a running container runtime [2]
- Images can be copied directly between registries, or pulled to a tarball and then pushed to another registry [2]

**Key concepts:**
- **OCI Compliant Artifacts**: Container images follow the Open Container Initiative (OCI) specification, which standardizes image formats and distribution [3]
- **Layered filesystem**: Container images are constructed from layered tarballs; each Dockerfile instruction produces a new layer [2]
- **Registry API**: Registries expose APIs (Docker Registry API V2 or OCI Distribution Spec) that tools use to push, pull, and manage images [2]

### Q2: What are air-gapped environments and why do organizations use them?

An air gap refers to the physical isolation of computer systems or networks so they cannot physically connect to other computers or the internet [4]. Air-gapped networks are networks that have been isolated from all external networks, including cloud and wifi [4].

**Why organizations use air-gapped environments:**
- **Government agencies**: Maintain confidentiality of state secrets, defense systems, and confidential source identities [4]
- **Financial institutions**: Protect transaction histories, passwords, and personally identifiable information (PII) from unauthorized access and data breaches [4]
- **Healthcare providers**: Secure patient records, protect research data, and ensure HIPAA compliance [4]
- **Critical infrastructure**: Power plants, water systems, air traffic control—keep industrial control systems safe from disruptions [4]
- **Research facilities**: Protect aerospace, pharmaceutical, and scientific innovations from industrial espionage [4]

**Types of air gapping:**
- **Physical air gap**: Complete physical disconnection—highest security but requires manual updates [4]
- **Logical air gap**: Software-based isolation with controlled access points [4]
- **Cloud-based air gap**: Isolated cloud environments with strict access controls [4]

**Benefits:**
- Network isolation from vulnerable networks [4]
- Ransomware protection—attackers cannot access air-gapped backups [4]
- Data loss security—last line of defense against cyberattacks [4]
- Enhanced encryption and access controls [4]

**Vulnerabilities:**
- Manual updating required—systems can become outdated [4]
- Human error with portable storage devices (USB drives) can introduce malware [4]
- Supply chain attacks (e.g., Stuxnet) can target air-gapped systems through pre-installed software [4]

### Q3: What problems does image mirroring solve in air-gapped environments?

Image mirroring addresses the fundamental challenge of deploying containerized applications in environments without internet connectivity [5].

**Problems solved:**

1. **Container image availability**: Kubernetes and container runtimes need to pull images to run workloads. Without mirroring, air-gapped clusters cannot access public registries [5]

2. **Deployment consistency**: The registry mirror option in Kubernetes allows images to be "magically" pulled from the local mirror. Developers don't need to maintain different versions of manifests or Helm values.yaml files for air-gapped vs. connected environments [5]

3. **Update management**: Organizations can control which images and versions are available in the air-gapped environment, enabling security scanning and approval workflows before images enter the secure zone [6]

4. **Bandwidth and latency**: Local mirrors provide faster image pulls and reduce dependency on external network connectivity [1]

5. **Security compliance**: Mirroring allows organizations to maintain a curated, approved set of images that have been scanned and verified before deployment [6]

6. **Operational continuity**: Even in DDIL (Denied, Disrupted, Intermittent, Limited) environments, workloads can continue operating with locally mirrored images [3]

### Q4: What are the common tools and approaches for image mirroring?

**Command-line tools:**

- **Skopeo**: Works with API V2 container registries, supports copying between registries, local directories, and OCI-layout directories. Can sync images, verify signatures, and delete images [2][7]
- **Crane**: Google's tool from go-containerregistry. Supports similar operations to skopeo but can also be used as a Go library. Useful for programmatic image manipulation [2]
- **oc-mirror**: Red Hat's OpenShift CLI plugin for mirroring OpenShift content. Provides centralized mirroring, maintains update paths, uses declarative configuration, and performs incremental mirroring [8]
- **Hauler**: Rancher Government's tool designed specifically for air-gapped environments. Single binary with zero dependencies, stores content as OCI compliant artifacts, includes embedded registry and fileserver [3]

**Key operations supported:**
- Copy images between registries [2]
- Pull images to tarballs for physical transfer [2]
- Push from tarballs to registries [2]
- List images and tags in registries [2]
- Delete images from registries [2]
- Sync entire repositories [7]

**Approaches:**

1. **Direct registry-to-registry mirroring**: When partial connectivity exists, copy directly from source to destination registry [8]

2. **Disk-based transfer (sneakernet)**: 
   - Mirror images to disk/tarball on connected system
   - Physically transfer media to air-gapped environment
   - Load images from disk to air-gapped registry [8][3]

3. **Incremental mirroring**: After initial sync, only download changed images to reduce transfer size [8]

### Summary

Container image mirroring is essential for running containerized workloads in air-gapped environments. It involves copying container images from public registries to private, local registries that can be accessed without internet connectivity. Air-gapped environments are used by organizations with strict security requirements—government, financial, healthcare, and critical infrastructure sectors—to protect sensitive data from cyber threats [4][5].

The key problems mirroring solves include: enabling container deployment without internet access, maintaining deployment consistency across connected and disconnected environments, controlling which images enter secure zones, and ensuring operational continuity [5][6]. Tools like skopeo, crane, oc-mirror, and Hauler provide various approaches to mirroring, from direct registry-to-registry copying to disk-based transfer for fully disconnected environments [2][3][8].

---

## Deeper Dive

### Subtopic 1: Implementation Patterns and Architecture

See `implementation-patterns-notes.md` for detailed notes.

### Subtopic 2: LLM and ML Model Considerations

See `llm-ml-model-notes.md` for detailed notes.

### Subtopic 3: Security and Compliance

See `security-compliance-notes.md` for detailed notes.
