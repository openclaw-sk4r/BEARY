# NIM Caching in Airgapped Environments — Notes

<!-- Notes for the NIM caching in airgapped environments topic. Organized by question, not source. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->
<!-- Citations are stored in whitepaper/NIM-caching-airgapped-envs-references.md -->

## General Understanding

<!-- Notes from initial research questions. One subsection per question. -->

### Q1: What is NVIDIA NIM and how does it package and serve AI models?

NVIDIA NIM (NVIDIA Inference Microservices) is a set of optimized cloud-native microservices designed to accelerate the deployment of generative AI models across cloud, data center, and GPU-accelerated workstations [1]. NIM abstracts away model inference internals such as execution engine and runtime operations, providing developers with OpenAI-compatible APIs [1].

**Two NIM Container Options:**
- **Multi-LLM compatible NIM**: A single container that enables deployment of a broad range of models, offering maximum flexibility. Supports models from NGC, HuggingFace, or local disk [1].
- **LLM-specific NIM**: Each container is focused on individual models or model families, offering maximum performance with pre-built, optimized engines for specific model/GPU combinations [1].

**Key Architecture Components:**
- NIMs are packaged as Docker containers on a per-model or per-model-family basis [1]
- Each NIM includes a runtime that runs on any NVIDIA GPU with sufficient GPU memory
- NIM leverages optimized inference engines (TensorRT-LLM, vLLM, SGLang) for each model and hardware setup [2]
- Model profiles define GPU-specific optimizations including tensor parallelism (TP) and pipeline parallelism (PP) configurations [3]

**Model Profiles:**
NIM uses model profiles that are tuned for specific NVIDIA GPUs, number of GPUs, precision, and other resources. NVIDIA produces model engines for several popular combinations. Profile names encode metadata like `tensorrt_llm-trtllm_buildable-bf16-tp8-pp2` indicating the engine, precision, and parallelism settings [3].

### Q2: What are the core challenges of deploying NIM in airgapped/disconnected environments?

Airgapped environments (also called air wall, air-gapping, or disconnected networks) present several challenges for NIM deployment [4][5]:

**Primary Challenges:**
1. **No NGC Registry Access**: Cannot pull NIM container images from nvcr.io directly [4]
2. **No Model Download**: Cannot download model artifacts from NGC or HuggingFace Hub at runtime [4]
3. **No License Validation**: Standard NGC API key validation requires internet connectivity [5]
4. **Certificate Management**: Proxy environments may require custom CA certificates [5]

**Two Deployment Options for Airgapped Systems:**
1. **Offline Cache Option**: Pre-download model profiles using `download-to-cache` command, then transfer the cache directory to the airgapped system [4]
2. **Local Model Directory Option**: Use `create-model-store` command to create a self-contained model repository that can be transferred [4]

**Critical Requirement**: When running in airgapped mode, do NOT provide the `NGC_API_KEY` environment variable, as this would cause the container to attempt network validation [4].

### Q3: How does NIM separate container images from model artifacts, and why is this separation important?

**The Separation Architecture:**
NIM deliberately separates the container image (runtime/inference engine) from the model artifacts (weights, engines, configurations) [3][6]:

- **Container Image**: Contains the NIM runtime, inference engines (TensorRT-LLM, vLLM), and serving infrastructure. Pulled from nvcr.io registry.
- **Model Artifacts**: Model weights, optimized TensorRT engines, tokenizers, and configuration files. Downloaded separately and cached.

**Cache Location:**
- Default cache path inside container: `/opt/nim/.cache` [3]
- Configurable via `NIM_CACHE_PATH` environment variable [3]
- HuggingFace models cached at: `${NIM_CACHE_PATH}/huggingface/hub` [3]

**Why Separation Matters:**

1. **Storage Efficiency**: Model artifacts (often 10-100+ GB) can be shared across multiple pod replicas via persistent volumes, avoiding redundant downloads [6]
2. **Startup Time**: Pre-cached models eliminate download time at container startup [6]
3. **Airgapped Compatibility**: Containers and models can be transferred separately using different mechanisms [4]
4. **Version Independence**: Container runtime can be updated independently of model artifacts
5. **Multi-Model Support**: Same container can serve different models by mounting different cache directories

**Cache Directory Structure** (observed from user reports):
```
/model-store/ngc/hub/models--nim--meta--llama3-8b-instruct/blobs/
```
The structure follows a HuggingFace-like hub pattern with model identifiers and blob storage [7].

### Q4: What is the role of Helm charts in NIM deployment and how do they interact with model caching?

**Helm Chart Purpose:**
NVIDIA provides the `nim-llm` Helm chart for Kubernetes deployments, encapsulating the complexity of NIM configuration [8]. The chart handles:
- Container deployment and GPU scheduling
- Secret management for NGC API keys and image pull credentials
- Persistent volume configuration for model caching
- Service exposure and ingress configuration
- Autoscaling and resource limits

**Key Helm Values for Caching:**

```yaml
image:
  repository: "nvcr.io/nim/meta/llama3-8b-instruct"
  tag: 1.0.3
model:
  ngcAPISecret: ngc-api  # Secret containing NGC_API_KEY
persistence:
  enabled: true
  size: 50Gi  # Must accommodate model size
imagePullSecrets:
  - name: ngc-secret  # For pulling from nvcr.io
```
[8]

**Persistence Configuration:**
- `persistence.enabled: true` creates a PVC for model caching
- Without persistence, every pod startup downloads the model to an `emptyDir` volume [8]
- For scaling, `ReadWriteMany` storage class (NFS, CephFS) is required so multiple pods can share the cache [8]

**Required Kubernetes Secrets:**
1. **Image Pull Secret** (`ngc-secret`): For pulling NIM container from nvcr.io
   ```bash
   kubectl create secret docker-registry ngc-secret \
     --docker-server=nvcr.io \
     --docker-username='$oauthtoken' \
     --docker-password=$NGC_API_KEY
   ```
2. **NGC API Secret** (`ngc-api`): For downloading model artifacts
   ```bash
   kubectl create secret generic ngc-api \
     --from-literal=NGC_API_KEY=$NGC_API_KEY
   ```
[8]

**NIM Operator Integration:**
The NIM Operator provides a `NIMCache` custom resource that manages model caching as a Kubernetes-native operation [6]:
- Creates caching jobs that download models to PVCs
- Supports automatic profile selection or specific profile caching
- Enables pre-caching before NIM service deployment
- Supports mirrored local model registries via `NIM_REPOSITORY_OVERRIDE` [9]

### Summary

NVIDIA NIM provides a containerized approach to AI model serving that deliberately separates the inference runtime (container image) from model artifacts (weights and engines). This separation is fundamental to airgapped deployment strategies.

**Key Themes:**
1. **Two-Component Architecture**: NIM containers contain the serving infrastructure while model artifacts are cached separately, enabling independent transfer and versioning.
2. **Profile-Based Optimization**: Model profiles encode GPU-specific optimizations; the correct profile must be cached for the target hardware.
3. **Helm + Operator Ecosystem**: Kubernetes deployments use Helm charts for basic deployment and the NIM Operator for advanced caching workflows.
4. **Airgapped Workflow**: Requires pre-downloading both container images (to private registry) and model artifacts (to transferable cache), then deploying without NGC_API_KEY.

**Agreements Across Sources:**
- All sources confirm the `/opt/nim/.cache` default cache path [3][6][7]
- Consistent guidance on not setting NGC_API_KEY in airgapped deployments [4][5]
- Universal recommendation for ReadWriteMany storage for multi-pod scaling [6][8]

---

## Deeper Dive

<!-- Notes from in-depth research questions. One subsection per question. -->

### Subtopic 1: Container Registry and Image Management

#### Q1: How do you mirror NIM container images to a private registry for airgapped use?

**Methods for Mirroring NIM Images:**

1. **Using `az acr import` (Azure):**
   ```bash
   az acr import \
     --name $ACR_NAME \
     --source nvcr.io/nim/$CONTAINER_AND_TAG \
     --image $CONTAINER_AND_TAG \
     --username '$oauthtoken' \
     --password $NGC_SECRET
   ```
   This directly imports from NGC to Azure Container Registry without needing local Docker [10].

2. **Using Skopeo (Recommended for airgapped):**
   ```bash
   skopeo login -u '$oauthtoken' -p "${NGC_API_KEY}" nvcr.io
   skopeo login -u "${USER}" -p "${PASSWORD}" "${REGISTRY}"
   skopeo copy --override-os linux --multi-arch all \
     docker://nvcr.io/nim/nvidia/llama-3.2-nemoretriever-300m-embed-v1:latest \
     "docker://${REGISTRY}/nim/nvidia/llama-3.2-nemoretriever-300m-embed-v1:latest"
   ```
   Skopeo can copy images between registries without requiring a local Docker daemon [11][12].

3. **Using Docker save/load:**
   ```bash
   # On connected machine
   docker pull nvcr.io/nim/meta/llama3-8b-instruct:latest
   docker save nvcr.io/nim/meta/llama3-8b-instruct:latest -o nim-llama3.tar
   
   # Transfer tar file to airgapped environment
   # On airgapped machine
   docker load -i nim-llama3.tar
   docker tag nvcr.io/nim/meta/llama3-8b-instruct:latest \
     private-registry.local/nim/meta/llama3-8b-instruct:latest
   docker push private-registry.local/nim/meta/llama3-8b-instruct:latest
   ```
   [13]

**Authentication to NGC:**
- Username is always `$oauthtoken` (literal string)
- Password is your NGC API key
- Registry URL is `nvcr.io` [10]

#### Q2: What tools and processes are used to transfer NIM images across airgap boundaries?

**Skopeo** is the recommended tool for registry-to-registry or registry-to-directory transfers [11]:
- Copies images without requiring a running Docker daemon
- Supports multi-architecture images with `--multi-arch all`
- Can sync entire registries: `skopeo sync --src docker --dest dir registry.example.com/busybox /media/usb`
- Supports authentication via `skopeo login` or `--src-creds`/`--dest-creds` flags

**For Kubernetes clusters (K3s example):**
1. Download images archive from releases
2. Use `docker image load` to import from tar
3. Use `docker tag` and `docker push` to retag and push to private registry
4. Configure `registries.yaml` to point to private registry [14]

**Image Size Considerations:**
NIM container images are large (10-30+ GB). Azure recommends enabling "artifact streaming" for faster cold starts with large AI workloads [10].

### Subtopic 2: Model Artifact Caching and Storage

#### Q1: How are NIM model artifacts downloaded, cached, and stored separately from containers?

**NIM Utility Commands for Caching:**

1. **`list-model-profiles`**: Lists all available profiles and their compatibility with current hardware [15]
   ```bash
   docker run --rm --runtime=nvidia --gpus=all \
     -e NGC_API_KEY=$NGC_API_KEY \
     $IMG_NAME list-model-profiles
   ```
   Output shows compatible profiles with their hash IDs (e.g., `6f437946f8efbca34997428528d69b08974197de157460cbe36c34939dc99edb`).

2. **`download-to-cache`**: Pre-downloads model profiles to the NIM cache [15]
   ```bash
   docker run -it --rm --gpus all \
     -e NGC_API_KEY \
     -v $LOCAL_NIM_CACHE:/opt/nim/.cache \
     $IMG_NAME download-to-cache -p <PROFILE_HASH>
   ```
   Options: `-p <hash>` for specific profile, `--all` for all profiles, `--lora` for LoRA profile.

3. **`create-model-store`**: Extracts cached profile to a standalone directory [15]
   ```bash
   docker run -it --rm --gpus all \
     -e NGC_API_KEY \
     -v $LOCAL_NIM_CACHE:/opt/nim/.cache \
     $IMG_NAME create-model-store -p <PROFILE_HASH> -m /model-store
   ```

**Cache Directory Structure:**
The cache follows a HuggingFace-like hub pattern:
```
/opt/nim/.cache/
├── ngc/
│   └── hub/
│       └── models--nim--meta--llama3-8b-instruct/
│           └── blobs/
```
[7]

#### Q2: What storage backends and persistent volume configurations are recommended for NIM model caching?

**Recommended Storage Classes:**

1. **ReadWriteMany (RWX)** storage is strongly recommended for production [16]:
   - Enables multiple pods to share the same cached model
   - Required for horizontal scaling and rolling upgrades
   - Examples: NFS, CephFS, AWS EFS, Azure Files

2. **Configuration in Helm values:**
   ```yaml
   persistence:
     enabled: true
     storageClass: "nfs"
     size: 50Gi
     accessModes:
       - ReadWriteMany
   ```
   [16]

3. **For single-node development:** Local Path Provisioner from Rancher Labs is sufficient [6].

**Storage Sizing:**
- Small models (8B parameters): 50-100 GB
- Large models (70B parameters): 200-300+ GB
- Account for multiple profiles if caching all variants

**NIM Operator PVC Management:**
The NIM Operator can automatically create PVCs via the `NIMCache` custom resource:
```yaml
spec:
  storage:
    pvc:
      create: true
      storageClass: 'nfs-client'
      size: "50Gi"
      volumeAccessMode: ReadWriteMany
```
[6]

#### Q3: How do you pre-populate model caches for airgapped deployments?

**Three-Phase Workflow (EDB/Enterprise Pattern)** [17]:

**Phase 1 - Build Cache (Connected Environment):**
1. Copy NIM images to private registry using skopeo
2. Run `list-model-profiles` to discover compatible profiles for target hardware
3. Run `download-to-cache` to download required profiles:
   ```bash
   docker run -it --rm --gpus all \
     -e NGC_API_KEY \
     -v ./nim-cache:/opt/nim/.cache \
     $IMG_NAME download-to-cache -p <PROFILE_HASH>
   ```

**Phase 2 - Upload to Object Storage:**
Upload the local cache directory to S3, GCS, or other object storage accessible from the airgapped cluster:
```bash
aws s3 sync ./nim-cache s3://bucket/model-cache/nim-llama3-8b/
```

**Phase 3 - Deploy in Airgapped Cluster:**
- Configure NIM to use `NIM_REPOSITORY_OVERRIDE` pointing to the object storage
- Mount the cache via PVC or init container
- Do NOT set `NGC_API_KEY` to prevent network validation attempts

**KServe-Specific Approach** [18]:
KServe requires PVCs mounted in ReadOnly mode. The cache must be pre-populated outside the InferenceService:
1. Create a PVC with ReadWriteMany access
2. Run a Job/Pod to download the cache to the PVC
3. Deploy InferenceService with the PVC mounted ReadOnly

### Subtopic 3: Helm Chart Configuration for Airgapped Deployments

#### Q1: What Helm chart values must be configured for NIM to work in airgapped environments?

**Critical Helm Values for Airgapped Deployment:**

```yaml
image:
  repository: "private-registry.local/nim/meta/llama3-8b-instruct"  # Private registry
  tag: "1.0.3"
  pullPolicy: IfNotPresent  # Avoid pulling from internet

imagePullSecrets:
  - name: private-registry-secret  # Secret for private registry auth

model:
  ngcAPISecret: ""  # Leave empty or omit for airgapped

persistence:
  enabled: true
  existingClaim: "pre-populated-nim-cache"  # Use pre-populated PVC

env:
  - name: NIM_REPOSITORY_OVERRIDE
    value: "s3://bucket/nim-models/"  # Or https:// or local path
  # Do NOT include NGC_API_KEY
```
[8][9]

**For NIM Operator with Mirrored Registry:**
```yaml
apiVersion: apps.nvidia.com/v1alpha1
kind: NIMService
metadata:
  name: meta-llama3-8b-instruct
spec:
  env:
    - name: "NGC_API_ENDPOINT"
      value: "<ARTIFACTORY-URL>/artifactory/api/nimmodel/<REPO>"
    - name: "NIM_REPOSITORY_OVERRIDE"
      value: "https://<server-name>:<port>/"
  image:
    repository: <PRIVATE-REGISTRY>/nim/meta/llama-3.1-8b-instruct
    pullSecrets:
      - ngc-secret
```
[9]

#### Q2: How do Helm charts handle model cache initialization and persistence?

**Default Behavior:**
- Without persistence, NIM downloads models to an `emptyDir` volume on every pod startup [8]
- With `persistence.enabled: true`, a PVC is created and models are cached persistently

**Scaling Considerations:**
- `statefulset.enabled: false` with `persistence.enabled: true` requires ReadWriteMany storage [8]
- For ReadWriteOnce storage, use StatefulSet mode where each replica gets its own PVC

**Init Container Pattern (KServe):**
Since KServe mounts PVCs as ReadOnly, a separate download Job must pre-populate the cache [18]:
```yaml
# Download Job
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

### Subtopic 4: Best Practices and Troubleshooting

#### Q1: What are industry best practices for NIM deployment in airgapped enterprise environments?

**Best Practices:**

1. **Separate Container and Model Transfer:**
   - Mirror container images to private registry using skopeo
   - Pre-download model profiles using `download-to-cache`
   - Transfer model cache via object storage or physical media [17]

2. **Profile Selection:**
   - Run `list-model-profiles` on hardware matching production
   - Cache only the profiles needed for your specific GPU configuration
   - Document profile hashes for reproducibility [15]

3. **Storage Architecture:**
   - Use ReadWriteMany storage (NFS, CephFS) for multi-pod deployments [16]
   - Size PVCs appropriately (50-300+ GB depending on model)
   - Consider artifact streaming for faster cold starts [10]

4. **Security:**
   - Never include NGC_API_KEY in airgapped deployments [4]
   - Use Kubernetes secrets for private registry credentials
   - Configure TLS termination and proper ingress routing [19]

5. **Validation:**
   - Test the complete workflow in a staging environment
   - Verify profile compatibility with target hardware before transfer
   - Use `nim-llm-check-cache-env` to verify cache accessibility [15]

#### Q2: What are common issues and troubleshooting steps for NIM caching in airgapped setups?

**Common Issues:**

1. **Image Pull Failures:**
   - Verify imagePullSecrets are correctly configured
   - Check private registry authentication
   - Ensure image tags match exactly [8]

2. **Model Download Timeouts:**
   - Increase `startupProbe.failureThreshold` in Helm values [8]
   - Pre-cache models to avoid download at startup
   - Check network connectivity to model repository

3. **Cache Permission Errors:**
   - NIM runs as non-root user (UID 1000 by default)
   - Ensure cache directory has correct permissions: `chown -R 1000:1000 /cache`
   - Use `spec.userID` in NIMCache to match [6]

4. **Profile Incompatibility:**
   - Run `list-model-profiles` to verify hardware compatibility
   - Ensure cached profile matches target GPU type and count
   - Check tensor parallelism requirements vs available GPUs [15]

5. **Missing config.json:**
   - HuggingFace models require `config.json` at root
   - For GGUF models, download config from full-precision repository [20]

6. **PVC Mount Issues:**
   - Verify StorageClass supports required access mode
   - Check PVC is bound before deploying NIM
   - For KServe, ensure PVC is pre-populated before InferenceService creation [18]

**Diagnostic Commands:**
```bash
# Check NIM cache environment
docker run --rm -v /path/to/cache:/opt/nim/.cache $IMG_NAME nim-llm-check-cache-env

# List available profiles
docker run --rm --gpus all -e NGC_API_KEY $IMG_NAME list-model-profiles

# Verify Kubernetes PVC
kubectl get pvc -n nim-service
kubectl describe pvc <pvc-name> -n nim-service
```

### Summary

The Deeper Dive research reveals a mature but complex ecosystem for airgapped NIM deployment. The workflow fundamentally requires treating container images and model artifacts as separate concerns that must be independently transferred and configured.

**Key Technical Insights:**

1. **Tooling:** Skopeo is the preferred tool for container image mirroring due to its daemon-less operation and multi-architecture support. NIM's built-in utilities (`download-to-cache`, `create-model-store`, `list-model-profiles`) handle model artifact management.

2. **Storage:** ReadWriteMany storage is essential for production deployments. The NIM Operator and Helm charts both support automatic PVC creation, but pre-population requires separate Jobs or manual intervention.

3. **Configuration:** The `NIM_REPOSITORY_OVERRIDE` environment variable is the key mechanism for redirecting model downloads to local/private sources. Critically, `NGC_API_KEY` must be omitted in airgapped deployments.

4. **Profiles:** Model profiles are hardware-specific optimizations. The correct profile must be identified using `list-model-profiles` on matching hardware, then cached and transferred. Profile hashes serve as unique identifiers.

**Connections to General Understanding:**
- The container/model separation (Q3 General) directly enables the two-phase transfer workflow
- Helm chart configuration (Q4 General) extends to airgapped-specific values like `NIM_REPOSITORY_OVERRIDE`
- The NIM Operator's `NIMCache` resource (Q4 General) supports mirrored registries via the same override mechanism

