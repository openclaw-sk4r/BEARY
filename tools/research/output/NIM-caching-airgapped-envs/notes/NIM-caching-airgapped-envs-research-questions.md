# NIM Caching in Airgapped Environments — Research Questions

<!-- Research questions for the NIM caching in airgapped environments topic. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->

## General Understanding

<!-- 3-4 general research questions to build foundational knowledge. Each question should have 3 search terms. -->

### Q1: What is NVIDIA NIM and how does it package and serve AI models?

**Search terms:**
- NVIDIA NIM architecture model serving
- NIM container model artifacts structure
- NVIDIA NIM inference microservices overview

### Q2: What are the core challenges of deploying NIM in airgapped/disconnected environments?

**Search terms:**
- NIM airgapped deployment challenges
- NVIDIA NIM offline installation requirements
- NIM disconnected environment deployment

### Q3: How does NIM separate container images from model artifacts, and why is this separation important?

**Search terms:**
- NIM model cache vs container image
- NVIDIA NIM model download separate from container
- NIM NGC model artifacts storage

### Q4: What is the role of Helm charts in NIM deployment and how do they interact with model caching?

**Search terms:**
- NIM Helm chart configuration
- NVIDIA NIM Kubernetes Helm deployment
- NIM Helm model cache PVC configuration

---

## Deeper Dive

<!-- 2-3 in-depth research questions per subtopic. HYPERPHAGIA mode encourages subtopics. -->

### Subtopic 1: Container Registry and Image Management

#### Q1: How do you mirror NIM container images to a private registry for airgapped use?

**Search terms:**
- NIM container image mirror private registry
- NGC container registry airgapped mirror
- NVIDIA NIM image pull offline registry

#### Q2: What tools and processes are used to transfer NIM images across airgap boundaries?

**Search terms:**
- docker save load airgapped transfer NIM
- skopeo copy NGC images private registry
- container image transfer airgapped Kubernetes

### Subtopic 2: Model Artifact Caching and Storage

#### Q1: How are NIM model artifacts downloaded, cached, and stored separately from containers?

**Search terms:**
- NIM model cache directory structure
- NGC CLI download model artifacts airgapped
- NIM LOCAL_NIM_CACHE environment variable

#### Q2: What storage backends and persistent volume configurations are recommended for NIM model caching?

**Search terms:**
- NIM model cache persistent volume Kubernetes
- NIM NFS storage model artifacts
- NVIDIA NIM storage best practices airgapped

#### Q3: How do you pre-populate model caches for airgapped deployments?

**Search terms:**
- NIM model cache pre-download offline
- NGC model download transfer airgapped
- NIM model artifacts bundle offline deployment

### Subtopic 3: Helm Chart Configuration for Airgapped Deployments

#### Q1: What Helm chart values must be configured for NIM to work in airgapped environments?

**Search terms:**
- NIM Helm values airgapped configuration
- NVIDIA NIM Helm chart private registry
- NIM Helm imagePullSecrets NGC offline

#### Q2: How do Helm charts handle model cache initialization and persistence?

**Search terms:**
- NIM Helm chart model cache init container
- NIM Helm PVC model persistence
- NVIDIA NIM Helm storage class configuration

### Subtopic 4: Best Practices and Troubleshooting

#### Q1: What are industry best practices for NIM deployment in airgapped enterprise environments?

**Search terms:**
- NIM enterprise airgapped deployment best practices
- NVIDIA NIM production offline deployment guide
- NIM disconnected Kubernetes deployment patterns

#### Q2: What are common issues and troubleshooting steps for NIM caching in airgapped setups?

**Search terms:**
- NIM model cache troubleshooting airgapped
- NIM container pull error private registry
- NVIDIA NIM offline deployment issues solutions

---

## Redundant Questions

<!-- Move any redundant questions here during review. -->

