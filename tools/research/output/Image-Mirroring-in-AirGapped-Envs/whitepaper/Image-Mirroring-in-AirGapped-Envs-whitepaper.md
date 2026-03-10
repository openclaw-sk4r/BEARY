# Image Mirroring in Air-Gapped Environments

<!-- See .windsurf/skills/whitepaper-writing/SKILL.md for whitepaper writing guidelines. -->

**Author:** Background Research Agent (BEARY)  
**Date:** 2026-03-02

---

## Abstract

This whitepaper examines container image mirroring in air-gapped environments—a critical capability for organizations deploying containerized workloads, including Large Language Models (LLMs), in isolated networks. We explore what image mirroring is, why air-gapped environments exist, the problems mirroring solves, and the tools and architectural patterns available for implementation. Special attention is given to the unique challenges of deploying large ML/LLM models in disconnected environments and the security considerations for maintaining image integrity and provenance. The findings indicate that while image mirroring has become significantly more accessible through mature tooling like Hauler, skopeo, crane, and oc-mirror, LLM deployments require additional architectural considerations—specifically separating model artifacts from container images and using S3-compatible object storage for large model weights.

## Introduction

Modern enterprise computing increasingly relies on containerized applications orchestrated by Kubernetes. However, many organizations—particularly in government, defense, financial services, healthcare, and critical infrastructure—operate in air-gapped environments where systems are intentionally isolated from external networks for security reasons [4]. This isolation creates a fundamental challenge: how do you deploy container workloads when the images they require exist only on public registries that cannot be accessed?

Image mirroring provides the answer. By copying container images from public registries to private, local registries within the air-gapped environment, organizations can run containerized workloads without compromising their security posture. This whitepaper addresses the following questions:

1. What is container image mirroring and how does it work?
2. What are air-gapped environments and why do organizations use them?
3. What problems does image mirroring solve?
4. What tools and architectural patterns exist for implementation?
5. What special considerations apply to LLM/ML model deployments?
6. How do organizations ensure security and compliance?

## Background

### Container Images and Registries

A container image is a self-contained, executable bundle containing everything required to run software in a well-defined runtime environment. Images consist of one or more tar archives plus a JSON manifest file describing the software and how to run it [2]. Container images follow the Open Container Initiative (OCI) specification, which standardizes image formats and distribution.

Container registries are services that store and distribute container images. Public registries like Docker Hub, quay.io, and the NVIDIA NGC Catalog host millions of images. Private registries can be deployed within an organization's infrastructure to host proprietary or mirrored images.

### Air-Gapped Environments

An air gap refers to the physical isolation of computer systems or networks so they cannot connect to other computers or the internet [4]. Air-gapped networks provide strong protection from network-based cyber threats but require all software and data to be transferred through controlled, often physical, means.

Organizations use air-gapped environments for several reasons:

- **Government agencies**: Protect classified information, state secrets, and defense systems [4]
- **Financial institutions**: Secure transaction data, customer PII, and trading systems [4]
- **Healthcare providers**: Maintain HIPAA compliance and protect patient records [4]
- **Critical infrastructure**: Isolate industrial control systems for power, water, and transportation [4]
- **Research facilities**: Protect intellectual property and sensitive research data [4]

## How Image Mirroring Works

### Core Concepts

Image mirroring copies container images from a source registry to a destination registry. The process involves:

1. **Authentication**: Obtaining credentials for both source and destination registries
2. **Image discovery**: Identifying which images and tags to mirror
3. **Layer transfer**: Copying the image manifest and all referenced layers
4. **Verification**: Ensuring the copied image matches the source

Tools like skopeo and crane interact with registries via the OCI Distribution Spec API without requiring a running container runtime [2]. This enables efficient registry-to-registry copying, as well as intermediate storage to tarballs for physical transfer.

### Mirroring Patterns

**Pattern 1: Partially Disconnected (Bastion Host)**

When controlled internet access exists via a bastion or jump host, mirroring tools can pull from public registries and push directly to internal mirror registries [8]. This is suitable for environments with limited but available connectivity.

**Pattern 2: Fully Disconnected (Sneakernet)**

For true air-gapped environments, images must be:
1. Mirrored to portable media (tarball/disk) on a connected system
2. Physically transferred across the air gap
3. Loaded from media to the air-gapped registry [8][3]

**Pattern 3: Incremental Mirroring**

After the initial sync (often the largest transfer), subsequent runs only download changed images. Tools like oc-mirror reference metadata from previous runs to determine deltas, significantly reducing transfer sizes for updates [8].

## Problems Solved by Image Mirroring

### Enabling Container Deployment

The most fundamental problem mirroring solves is enabling container deployment in environments without internet access. Kubernetes and container runtimes must pull images to run workloads—without a local registry containing those images, deployment is impossible [5].

### Deployment Consistency

The registry mirror option in Kubernetes allows images to be transparently pulled from local mirrors. Developers and operators don't need to maintain different versions of manifests or Helm values.yaml files for air-gapped versus connected environments [5]. The same deployment artifacts work in both contexts.

### Security and Compliance

Mirroring enables organizations to:
- Control which images enter the secure environment
- Scan images for vulnerabilities before admission
- Implement approval workflows for new images
- Maintain audit trails of all image transfers [6]

### Operational Continuity

In DDIL (Denied, Disrupted, Intermittent, Limited) environments, locally mirrored images ensure workloads continue operating regardless of external network conditions [3].

### Bandwidth and Performance

Local mirrors provide faster image pulls and eliminate dependency on external network bandwidth and latency [1].

## Tools for Image Mirroring

### Skopeo

Skopeo works with API V2 container registries and supports copying between registries, local directories, and OCI-layout directories. Key capabilities include:
- Registry-to-registry copying
- Pull to tarball / push from tarball
- Image signature verification
- Repository synchronization [2][7]

### Crane

Google's crane tool (from go-containerregistry) provides similar functionality to skopeo but can also be used as a Go library for programmatic image manipulation. This makes it valuable for building custom tooling [2].

### oc-mirror

Red Hat's OpenShift CLI plugin provides enterprise features for mirroring OpenShift content:
- Declarative image set configuration (YAML)
- Maintains update paths for platform and operators
- Incremental mirroring with metadata tracking
- Image pruning for removed content
- Generates cluster configuration artifacts [8]

### Hauler

Rancher Government's Hauler is purpose-built for air-gapped environments:
- Single binary with zero dependencies
- Stores content as OCI Compliant Artifacts
- Supports images, Helm charts, and files
- Includes embedded registry (port 5000) and fileserver (port 8080)
- Stores attestations, SBOMs, and signatures alongside images [3]

Hauler's workflow:
```
# Connected side
hauler store add image docker.io/library/nginx:latest
hauler store save --filename haul.tar.zst

# Transfer haul.tar.zst across air gap

# Disconnected side
hauler store load haul.tar.zst
hauler store serve registry  # Serves on port 5000
```

## LLM and ML Model Considerations

### The Size Challenge

Large Language Models present unique challenges for air-gapped deployment. While standard container images are typically megabytes to a few gigabytes, LLM model weights can span hundreds of gigabytes [9]. Attempting to package LLMs directly into container images or load them into container registries often fails catastrophically—"You can wait 30–45 minutes, and your reward is an out-of-disk failure" that can take out cluster worker nodes [9].

### Recommended Architecture

The solution is to separate model artifacts from container images:

1. **Container images**: Package only the serving infrastructure (VLLM, inference engines, APIs)
2. **Model artifacts**: Store separately in S3-compatible object storage (e.g., MinIO)
3. **Runtime mounting**: Models are loaded from object storage at container startup [9]

H2O.ai's production architecture demonstrates this pattern:
- MinIO stores model artifacts (tar files of safetensors and metadata)
- Model Hub provides a Hugging Face-compatible REST API
- VLLM serves inference requests, loading models from Model Hub
- Models are "sideloaded" using the MinIO client (mc) rather than going through container registries [9]

### NVIDIA NIM for Air-Gapped LLMs

NVIDIA NIM supports air-gapped LLM deployment through two approaches:

1. **Offline Cache Option**: Use `download-to-cache` to pre-download model profiles on a connected system, then transfer the cache directory to the air-gapped environment [10]

2. **Local Model Directory Option**: Use `create-model-store` to create a model repository that can be mounted at runtime [10]

After transferring to the air-gapped environment, containers run without NGC_API_KEY or HF_TOKEN—no external credentials required [10].

### Best Practices for LLM Deployment

1. **Differentiate by size**: Models under ~1GB can be containerized; larger models should be stored externally [9]
2. **Design for air-gapping from the start**: All artifacts must be deliverable offline, including GPU drivers [9]
3. **Plan for updates**: Build CI/CD pipelines supporting model promotion across dev/test/prod environments [9]
4. **Expose serving parameters**: VLLM and other engines have many tunable parameters affecting performance [9]

## Security and Compliance

### Supply Chain Security

Container images are often composed of layers from multiple sources, making it difficult to verify content without cryptographic measures [11]. Image signing addresses this by adding digital signatures that verify:
- The image hasn't been tampered with (integrity)
- The image comes from a trusted source (provenance)

### Image Signing

The standard approach uses asymmetric cryptography:
1. Create a private/public key pair
2. Sign images with the private key
3. Distribute the public key for verification
4. Container runtimes verify signatures before running images [11]

Tools like Cosign (Sigstore), Notation (Notary Project), and Docker Content Trust provide signing capabilities. Hauler preserves attestations, SBOMs, and signatures when mirroring, enabling verification in disconnected environments [3].

### Air-Gap Specific Considerations

- **Offline verification**: Air-gapped environments need local copies of public keys and trust anchors
- **Certificate management**: Time-limited certificates or local CRL copies address revocation checking
- **Physical media security**: USB drives and portable media can introduce malware—implement scanning and controls [4]
- **Supply chain attacks**: Pre-installed software can be compromised (e.g., Stuxnet)—verify all software sources [4]

## Discussion

Image mirroring in air-gapped environments has matured significantly. Tools like Hauler have "drastically improved the experience of deploying an air gapped container registry mirror" [5], and the standardization of registry mirror options in Kubernetes flavors enables transparent image sourcing without maintaining separate manifests for connected and disconnected environments.

However, LLM deployments introduce new complexity. The fundamental insight from production deployments like H2O.ai's is that large models should not go through container registries at all—they should be sideloaded into object storage and mounted at runtime [9]. This architectural pattern is essential for anyone implementing LLMs in air-gapped environments.

Security remains paramount. While air-gapping provides strong network isolation, it doesn't eliminate all threats. Organizations must implement image signing, maintain verification infrastructure locally, and carefully control physical media that crosses the air gap.

## Conclusion

Image mirroring is essential infrastructure for running containerized workloads in air-gapped environments. The technology has matured to the point where multiple robust tools exist—skopeo, crane, oc-mirror, and Hauler—each with different strengths for different use cases.

For LLM deployments specifically, the key takeaways are:

1. **Separate models from containers**: Store large model weights in S3-compatible object storage, not container registries
2. **Use purpose-built tooling**: NVIDIA NIM provides specific air-gap deployment options for LLMs
3. **Design for air-gapping from the start**: Retrofitting is much harder than building with isolation in mind
4. **Maintain security posture**: Image signing, verification, and controlled media transfer are essential

The problems image mirroring solves—enabling deployment, maintaining consistency, ensuring security compliance, and providing operational continuity—make it a foundational capability for any organization operating containerized workloads in isolated environments.

## References

See `Image-Mirroring-in-AirGapped-Envs-references.md` for the full bibliography.

In-text citations:
- [1] IBM MAS CLI - Image Mirroring Guide
- [2] D2iQ Engineering - Skopeo and Crane Tools
- [3] Rancher Government - Hauler for Air-Gapped Environments
- [4] IBM - What is an Air Gap?
- [5] SEI/CMU - Kubernetes in the Air Gap
- [6] Red Hat - MicroShift Mirror Registry Documentation
- [7] GitHub - Skopeo Repository
- [8] Red Hat - oc-mirror Plugin Documentation
- [9] Replicated/H2O.ai - Distributing AI Models to Self-Hosted Environments
- [10] NVIDIA - NIM Air Gap Deployment Guide
- [11] Aqua Security - Container Image Signing Guide
