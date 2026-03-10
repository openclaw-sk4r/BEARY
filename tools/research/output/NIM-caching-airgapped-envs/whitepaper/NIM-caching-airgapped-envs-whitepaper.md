# NIM Caching in Airgapped Environments

**Author:** Background Research Agent (BEARY)
**Date:** 2026-03-02

---

## Abstract

This whitepaper provides a technical deep-dive into NVIDIA NIM (NVIDIA Inference Microservices) caching workflows for airgapped environments where clusters cannot reach NGC (NVIDIA GPU Cloud) to download container images or model artifacts. We examine the fundamental architecture that separates NIM container images from model artifacts, explain why this separation is critical for airgapped deployments, and provide step-by-step guidance on mirroring containers, pre-populating model caches, configuring Helm charts, and troubleshooting common issues. The research draws from official NVIDIA documentation, enterprise deployment guides, and community troubleshooting forums to present a comprehensive picture of industry practices.

## Introduction

Enterprise environments increasingly require AI inference capabilities in airgapped or disconnected networks—environments with no internet connectivity due to security, compliance, or operational requirements. NVIDIA NIM provides optimized inference microservices for deploying large language models (LLMs) and other AI models, but its default workflow assumes network access to NGC for pulling container images and downloading model artifacts at runtime [1].

This whitepaper addresses the following questions relevant to deploying NIM in airgapped environments:

1. How does NIM's architecture separate containers from model artifacts, and why does this matter?
2. What tools and processes transfer NIM components across airgap boundaries?
3. How do Helm charts and the NIM Operator configure caching for offline use?
4. What are the storage requirements and best practices for model caching?
5. What common issues arise and how are they resolved?

The intended audience is MLOps engineers familiar with Kubernetes, Docker, and basic AI/ML concepts who need to understand the technical details of NIM airgapped deployment.

## Background

### NVIDIA NIM Architecture

NVIDIA NIM is a set of optimized cloud-native microservices designed to accelerate the deployment of generative AI models [1]. NIM abstracts away model inference internals—execution engines, runtime operations, and hardware optimizations—while providing developers with OpenAI-compatible APIs.

**Two NIM Container Options:**

- **LLM-specific NIM**: Containers focused on individual models or model families, offering maximum performance with pre-built, optimized engines for specific model/GPU combinations [1].
- **Multi-LLM compatible NIM**: A single container enabling deployment of a broad range of models from NGC, HuggingFace, or local disk, offering maximum flexibility [1].

### Model Profiles

NIM uses **model profiles**—pre-optimized configurations tuned for specific NVIDIA GPUs, GPU counts, precision levels, and inference engines. Profile names encode this metadata; for example, `tensorrt_llm-a100-bf16-tp4-throughput` indicates a TensorRT-LLM engine optimized for A100 GPUs with BF16 precision and 4-way tensor parallelism [3].

Each profile is identified by a unique hash (e.g., `6f437946f8efbca34997428528d69b08974197de157460cbe36c34939dc99edb`). When NIM starts, it selects the optimal profile for the detected hardware and downloads the corresponding model artifacts from NGC [15].

### The Airgapped Challenge

In airgapped environments, NIM faces two fundamental obstacles [4][5]:

1. **No container registry access**: Cannot pull NIM images from `nvcr.io`
2. **No model artifact download**: Cannot fetch model profiles from NGC at runtime

The solution requires pre-transferring both components and configuring NIM to operate without network validation.

## Container Registry and Image Management

### Mirroring NIM Images to Private Registries

NIM container images must be copied from NGC (`nvcr.io`) to a private registry accessible within the airgapped environment. Three primary methods exist:

**1. Skopeo (Recommended)**

Skopeo copies images between registries without requiring a local Docker daemon, making it ideal for automated pipelines [11]:

```bash
# Authenticate to both registries
skopeo login -u '$oauthtoken' -p "${NGC_API_KEY}" nvcr.io
skopeo login -u "${USER}" -p "${PASSWORD}" "${PRIVATE_REGISTRY}"

# Copy image with multi-architecture support
skopeo copy --override-os linux --multi-arch all \
  docker://nvcr.io/nim/meta/llama3-8b-instruct:1.0.3 \
  "docker://${PRIVATE_REGISTRY}/nim/meta/llama3-8b-instruct:1.0.3"
```

**2. Cloud Provider Import (Azure Example)**

Cloud registries often support direct import from NGC [10]:

```bash
az acr import \
  --name $ACR_NAME \
  --source nvcr.io/nim/meta/llama3-8b-instruct:1.0.3 \
  --image nim/meta/llama3-8b-instruct:1.0.3 \
  --username '$oauthtoken' \
  --password $NGC_API_KEY
```

**3. Docker Save/Load**

For physical media transfer [13]:

```bash
# Connected environment
docker pull nvcr.io/nim/meta/llama3-8b-instruct:1.0.3
docker save nvcr.io/nim/meta/llama3-8b-instruct:1.0.3 -o nim-llama3.tar

# Airgapped environment (after physical transfer)
docker load -i nim-llama3.tar
docker tag nvcr.io/nim/meta/llama3-8b-instruct:1.0.3 \
  private-registry.local/nim/meta/llama3-8b-instruct:1.0.3
docker push private-registry.local/nim/meta/llama3-8b-instruct:1.0.3
```

### Authentication Notes

NGC authentication uses a fixed username `$oauthtoken` (literal string) with your NGC API key as the password [10]. This applies to both container pulls and model artifact downloads.

### Image Size Considerations

NIM container images are large (10-30+ GB). For cloud deployments, consider enabling artifact streaming or similar technologies to reduce cold start times [10].

## Model Artifact Caching and Storage

### Model vs Container Separation

NIM deliberately separates the container image (runtime/inference engine) from model artifacts (weights, optimized engines, tokenizers) [3][6]. This separation is fundamental to airgapped deployment:

| Component | Contents | Transfer Method |
|-----------|----------|-----------------|
| Container Image | NIM runtime, inference engines (TensorRT-LLM, vLLM, SGLang), serving infrastructure | Registry mirroring |
| Model Artifacts | Model weights, optimized TensorRT engines, tokenizers, config files | Cache pre-population |

**Why Separation Matters:**

1. **Storage Efficiency**: Model artifacts (50-300+ GB) can be shared across multiple pod replicas via persistent volumes [6]
2. **Startup Time**: Pre-cached models eliminate download delays at container startup
3. **Independent Versioning**: Container runtime can be updated without re-downloading models
4. **Airgapped Compatibility**: Each component uses different transfer mechanisms

### NIM Utility Commands

NIM containers include built-in utilities for cache management [15]:

**`list-model-profiles`**: Discovers available profiles and their hardware compatibility:

```bash
docker run --rm --runtime=nvidia --gpus=all \
  -e NGC_API_KEY=$NGC_API_KEY \
  $IMG_NAME list-model-profiles
```

Output categorizes profiles as "Compatible with system," "With LoRA support," or "Incompatible with system."

**`download-to-cache`**: Pre-downloads model profiles:

```bash
docker run -it --rm --gpus all \
  -e NGC_API_KEY \
  -v $LOCAL_NIM_CACHE:/opt/nim/.cache \
  $IMG_NAME download-to-cache -p <PROFILE_HASH>
```

Options include `-p <hash>` for specific profiles, `--all` for all profiles, or `--lora` for LoRA-enabled profiles.

**`create-model-store`**: Extracts a cached profile to a standalone directory:

```bash
docker run -it --rm --gpus all \
  -e NGC_API_KEY \
  -v $LOCAL_NIM_CACHE:/opt/nim/.cache \
  $IMG_NAME create-model-store -p <PROFILE_HASH> -m /model-store
```

### Cache Directory Structure

The NIM cache follows a HuggingFace-like hub pattern [7]:

```
/opt/nim/.cache/
├── ngc/
│   └── hub/
│       └── models--nim--meta--llama3-8b-instruct/
│           └── blobs/
```

The default cache path is `/opt/nim/.cache`, configurable via the `NIM_CACHE_PATH` environment variable [3].

### Storage Backends and Persistent Volumes

**Recommended: ReadWriteMany (RWX) Storage**

For production Kubernetes deployments, RWX storage (NFS, CephFS, AWS EFS, Azure Files) is strongly recommended [16]:

- Enables multiple pods to share the same cached model
- Required for horizontal scaling and rolling upgrades
- Avoids redundant downloads across replicas

**Helm Configuration:**

```yaml
persistence:
  enabled: true
  storageClass: "nfs"
  size: 50Gi
  accessModes:
    - ReadWriteMany
```

**Storage Sizing Guidelines:**

| Model Size | Recommended PVC Size |
|------------|---------------------|
| 8B parameters | 50-100 GB |
| 70B parameters | 200-300 GB |
| Multiple profiles | Add 50-100% overhead |

### Pre-populating Model Caches

The enterprise workflow for airgapped cache preparation follows three phases [12][17]:

**Phase 1: Build Cache (Connected Environment)**

1. Mirror NIM container images to private registry
2. Run `list-model-profiles` on hardware matching production to identify compatible profiles
3. Run `download-to-cache` to download required profiles locally

**Phase 2: Transfer Cache**

Upload the local cache directory to object storage or transfer via physical media:

```bash
aws s3 sync ./nim-cache s3://bucket/model-cache/nim-llama3-8b/
```

**Phase 3: Deploy in Airgapped Cluster**

Configure NIM to use `NIM_REPOSITORY_OVERRIDE` pointing to the transferred cache location. Critically, do NOT set `NGC_API_KEY` to prevent network validation attempts [4].

## Helm Chart Configuration

### Required Values for Airgapped Deployments

The `nim-llm` Helm chart requires specific configuration for airgapped operation [8][9]:

```yaml
image:
  repository: "private-registry.local/nim/meta/llama3-8b-instruct"
  tag: "1.0.3"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: private-registry-secret

model:
  ngcAPISecret: ""  # Leave empty for airgapped

persistence:
  enabled: true
  existingClaim: "pre-populated-nim-cache"

env:
  - name: NIM_REPOSITORY_OVERRIDE
    value: "s3://bucket/nim-models/"
  # Do NOT include NGC_API_KEY
```

**Key Configuration Points:**

1. **`image.repository`**: Must point to your private registry
2. **`imagePullSecrets`**: Required for private registry authentication
3. **`model.ngcAPISecret`**: Leave empty or omit entirely
4. **`NIM_REPOSITORY_OVERRIDE`**: Redirects model downloads to local/private source
5. **`persistence.existingClaim`**: Use a pre-populated PVC

### NIM Operator Configuration

For NIM Operator deployments with mirrored registries [9]:

```yaml
apiVersion: apps.nvidia.com/v1alpha1
kind: NIMService
metadata:
  name: meta-llama3-8b-instruct
spec:
  env:
    - name: NIM_REPOSITORY_OVERRIDE
      value: "https://<mirror-server>:<port>/"
  image:
    repository: <PRIVATE-REGISTRY>/nim/meta/llama-3.1-8b-instruct
    pullSecrets:
      - ngc-secret
  storage:
    pvc:
      create: true
      storageClass: 'nfs-client'
      size: "50Gi"
      volumeAccessMode: ReadWriteMany
```

### Model Cache Initialization and Persistence

**Default Behavior:**

Without persistence enabled, NIM downloads models to an `emptyDir` volume on every pod startup—unacceptable for airgapped environments [8].

**Scaling Considerations:**

- `statefulset.enabled: false` with `persistence.enabled: true` requires ReadWriteMany storage
- For ReadWriteOnce storage, use StatefulSet mode where each replica gets its own PVC

**KServe-Specific Pattern:**

KServe requires PVCs mounted in ReadOnly mode, necessitating a separate cache population step [18]:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nim-cache-download
spec:
  template:
    spec:
      containers:
        - name: downloader
          image: private-registry/nim/meta/llama3-8b-instruct:latest
          args: ["download-to-cache", "-p", "<PROFILE_HASH>"]
          volumeMounts:
            - name: cache
              mountPath: /opt/nim/.cache
      volumes:
        - name: cache
          persistentVolumeClaim:
            claimName: nim-cache-pvc
```

## Best Practices and Troubleshooting

### Enterprise Deployment Patterns

**1. Separate Container and Model Transfer**

Treat container images and model artifacts as independent concerns with different transfer mechanisms [17].

**2. Profile Selection**

- Run `list-model-profiles` on hardware matching production
- Cache only profiles needed for your specific GPU configuration
- Document profile hashes for reproducibility

**3. Storage Architecture**

- Use ReadWriteMany storage for multi-pod deployments [16]
- Size PVCs appropriately for model size plus overhead
- Consider artifact streaming for faster cold starts [10]

**4. Security**

- Never include `NGC_API_KEY` in airgapped deployments [4]
- Use Kubernetes secrets for private registry credentials
- Configure TLS termination and proper ingress routing [19]

**5. Validation**

- Test the complete workflow in a staging environment
- Verify profile compatibility before transfer
- Use `nim-llm-check-cache-env` to verify cache accessibility [15]

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Image pull failures | Missing or incorrect imagePullSecrets | Verify secret configuration and registry authentication |
| Model download timeouts | Startup probes failing before download completes | Increase `startupProbe.failureThreshold` or pre-cache models |
| Cache permission errors | NIM runs as non-root (UID 1000) | `chown -R 1000:1000 /cache` or set `spec.userID` |
| Profile incompatibility | Cached profile doesn't match hardware | Run `list-model-profiles` to verify; cache correct profile |
| Missing config.json | HuggingFace model missing configuration | Download config.json from full-precision repository [20] |
| PVC mount failures | StorageClass doesn't support access mode | Verify StorageClass capabilities; use appropriate mode |

**Diagnostic Commands:**

```bash
# Verify cache environment
docker run --rm -v /path/to/cache:/opt/nim/.cache $IMG_NAME nim-llm-check-cache-env

# List profiles on target hardware
docker run --rm --gpus all -e NGC_API_KEY $IMG_NAME list-model-profiles

# Check Kubernetes PVC status
kubectl get pvc -n nim-service
kubectl describe pvc <pvc-name> -n nim-service
```

## Discussion

The research reveals a mature but complex ecosystem for airgapped NIM deployment. The fundamental insight is that NIM's architecture—separating container images from model artifacts—is both the source of complexity and the enabler of airgapped operation.

**Architectural Trade-offs:**

The separation allows independent versioning and efficient storage sharing, but requires managing two distinct transfer workflows. Organizations must maintain both a private container registry and a model artifact repository (object storage, NFS, or similar).

**Tooling Ecosystem:**

NVIDIA provides comprehensive utilities (`download-to-cache`, `create-model-store`, `list-model-profiles`) that handle the model artifact side, while standard container tools (Skopeo, Docker) handle image mirroring. The NIM Operator adds Kubernetes-native abstractions but introduces additional complexity.

**Storage as a Critical Dependency:**

ReadWriteMany storage emerges as a near-universal requirement for production deployments. Organizations without existing RWX infrastructure (NFS, CephFS) face significant setup overhead.

**Configuration Sensitivity:**

The requirement to omit `NGC_API_KEY` in airgapped deployments is counterintuitive and a common source of errors. The system attempts network validation when the key is present, causing failures in disconnected environments.

## Conclusion

Deploying NVIDIA NIM in airgapped environments requires understanding and managing the separation between container images and model artifacts. The key steps are:

1. **Mirror container images** to a private registry using Skopeo or equivalent tools
2. **Pre-download model profiles** using NIM's `download-to-cache` utility on hardware matching production
3. **Transfer the cache** via object storage or physical media
4. **Configure Helm/Operator** with `NIM_REPOSITORY_OVERRIDE` pointing to local sources and without `NGC_API_KEY`
5. **Use ReadWriteMany storage** for production multi-pod deployments

**Open Questions:**

- Automation of profile discovery and caching across heterogeneous GPU fleets
- Lifecycle management for model updates in airgapped environments
- Integration with enterprise artifact management systems (JFrog Artifactory, etc.)

The workflow is well-documented but requires careful attention to detail. Organizations should validate the complete process in staging environments before production deployment.

## References

See NIM-caching-airgapped-envs-references.md for the full bibliography.

