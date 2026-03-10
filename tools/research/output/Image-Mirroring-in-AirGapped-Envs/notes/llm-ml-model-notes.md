# Image Mirroring in AirGapped Environments — LLM and ML Model Notes

<!-- Notes for the LLM and ML Model Considerations subtopic of Image Mirroring in AirGapped Environments. Organized by question, not source. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->
<!-- Citations are stored in whitepaper/Image-Mirroring-in-AirGapped-Envs-references.md -->

## Questions

### Q1: What are the specific challenges of mirroring large ML/LLM model images in air-gapped environments?

**Size challenges:**
- LLM models often span hundreds of gigabytes, making them unsuitable for container packaging [9]
- Models under ~1GB can be packaged directly into Docker containers, but large models require different strategies [9]
- Trying to load LLMs directly into container registries can cause failures: "You can wait 30–45 minutes, and your reward is an out-of-disk failure" that can take out worker nodes [9]

**Transfer challenges:**
- Physical transfer of multi-hundred-gigabyte model files across air gaps is time-consuming
- Network bandwidth limitations even in partially connected environments
- Storage requirements on both connected and disconnected sides

**Registry limitations:**
- Container registries are not designed for very large artifacts [9]
- Registry storage can be overwhelmed by large model files
- Layer deduplication doesn't help much with model weights (unique binary data)

**NVIDIA NIM approach for air-gapped LLM deployment:**
- Supports serving models with no internet connection and no connection to NGC registry or Hugging Face Hub [10]
- Two options:
  1. **Offline Cache Option**: Use `download-to-cache` to pre-download model profiles, then transfer cache to air-gapped system [10]
  2. **Local Model Directory Option**: Use `create-model-store` to create a model repository, then mount at runtime [10]
- After downloading, do NOT provide NGC_API_KEY or HF_TOKEN—run without external credentials [10]

### Q2: What best practices exist for managing LLM container images in disconnected environments?

**Best Practice 1: Separate model artifacts from container images**
- Store large model artifacts externally (e.g., in MinIO/S3-compatible storage) [9]
- Mount models dynamically at runtime rather than baking into containers [9]
- Models bundled as tar files since individual files are under 10GB but collectively very large [9]

**Best Practice 2: Use S3-compatible object storage**
- H2O.ai architecture uses MinIO for storing model artifacts [9]
- S3 is widely adopted in enterprise environments, offering portability [9]
- Allows customers to swap in their own storage solutions [9]
- Model Hub reads from MinIO, VLLM reads from Model Hub [9]

**Best Practice 3: Sideload models using CLI tools**
- Use MinIO client (mc) to upload model files directly into object storage [9]
- This "sideloads" model bits without going through container registry [9]
- Avoids registry bloat and simplifies deployment [9]

**Best Practice 4: Differentiate strategies by model size**
- Models < 1GB: Package directly into Docker containers for tighter integration with software versioning [9]
- Large models (LLMs): Store separately, prune extraneous formats, keep only safetensors and relevant metadata [9]

**Best Practice 5: Design for air-gapping from the beginning**
- All software and artifacts must be deliverable offline, including drivers and Helm charts [9]
- For GPU support, choose between:
  - Full NVIDIA operator (requires internet for dynamic asset pulls)
  - Pre-installed driver setup with partial operator components [9]

**Best Practice 6: Plan for model updates and dev/prod parity**
- Customers typically have multiple environments (dev moves faster than prod) [9]
- Include environment-aware configuration (dev/test/prod values.yaml overrides) [9]
- Build CI/CD pipelines supporting model promotion across clusters [9]
- Allow side-by-side deployment of multiple model versions for A/B testing or rollback [9]

**Best Practice 7: Expose model-serving parameters**
- VLLM and other inference engines have dozens of runtime parameters affecting performance [9]
- Expose parameters in configurable, documented way [9]
- Set reasonable defaults for context length, kv cache, batch size [9]
- Provide profiling/logging to tune performance over time [9]

**H2O.ai Reference Architecture:**
- Replicated Embedded Cluster: Self-contained Kubernetes environment
- KOTS: Manages application components via Helm-based installs
- Keycloak: Authentication and authorization
- MinIO: S3-compatible object storage for model artifacts
- Model Hub: Hugging Face-compatible REST API for serving LLMs
- VLLM: GPU-optimized model server loading weights from local storage
- Enterprise GPTe: Agentic and RAG-based LLM capabilities [9]

---

## Summary

Managing LLM container images in air-gapped environments requires fundamentally different approaches than standard container workloads due to the massive size of model weights (often hundreds of gigabytes). The key insight is to separate model artifacts from container images—store models in S3-compatible object storage like MinIO and mount them at runtime rather than packaging into containers [9].

NVIDIA NIM provides specific tooling for air-gapped LLM deployment through offline cache and local model directory options [10]. H2O.ai's production architecture demonstrates a complete pattern: use lightweight containers for the serving infrastructure while sideloading model files directly into object storage, with Model Hub and VLLM providing the serving layer [9]. Organizations should design for air-gapping from the start, differentiate strategies by model size, and plan for the operational complexity of model updates across multiple environments [9].
