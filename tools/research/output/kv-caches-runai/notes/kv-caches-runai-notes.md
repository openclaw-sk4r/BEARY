# KV Caches and RunAI — Notes

<!-- Notes for the kv-caches-runai topic. Organized by question, not source. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->
<!-- Citations are stored in whitepaper/kv-caches-runai-references.md -->

## General Understanding

<!-- Notes from initial research questions. One subsection per question. -->

### Q1: What are KV caches in the context of LLM inference, and why are they important for performance?

KV (Key-Value) caches store intermediate key and value computations from the attention mechanism for reuse during inference, resulting in substantial speed-ups when generating text [1]. During autoregressive text generation, LLMs generate one token at a time. Without caching, each new token requires recomputing attention scores for all previous tokens, leading to redundant O(n²) computation [1][2].

**How KV Caching Works:**
- During the first forward pass (prefill), the model computes and stores key/value vectors for all input tokens [2]
- For each subsequent token generated, only the new token's K/V vectors are computed [1]
- Previously computed vectors are retrieved from cache, avoiding recomputation [2]
- This reduces per-step complexity from O(n²) to O(n) [1]

**Performance Impact:**
- Simple implementations show ~5x speed-up even with small 124M parameter models [1]
- Benefits increase with longer sequences and larger models [1]

**Memory Considerations:**
KV cache memory grows linearly with batch size and sequence length [3]. The formula for KV cache size per token is:
`2 × 2 × n_layers × n_heads × head_dim` bytes (for FP16) [4]

For Llama-2-7B, this amounts to ~0.5MB per token; for BLOOM-176B, ~4MB per token [3]. A 10,000 token context for Llama-2-7B requires ~5GB just for KV cache [4].

**Trade-offs:**
- **Good:** Computational efficiency increases dramatically
- **Bad:** Memory usage increases linearly with sequence length, potentially exceeding model weight memory for long contexts [1][3]

### Q2: How do inference serving frameworks (vLLM, TensorRT-LLM, TGI) implement and manage KV caches?

**vLLM and PagedAttention:**
vLLM introduced PagedAttention, inspired by OS virtual memory paging [5][6]. Key innovations:
- Allocates fixed-size memory blocks that can be stored non-contiguously
- On-demand allocation eliminates internal memory fragmentation
- Same-size blocks eliminate external memory fragmentation
- Achieves near-zero waste (<4%) vs. 80% waste in naive implementations [3]
- Enables flexible KV cache sharing within and across requests [5]

vLLM also supports Automatic Prefix Caching, which reuses KV cache blocks across requests with shared prefixes (e.g., system prompts) [6].

**TensorRT-LLM:**
TensorRT-LLM provides fine-grained KV cache control [7]:
- **Priority-based eviction:** Users can set retention priorities for token ranges (e.g., keep system prompts longer)
- **Duration-based retention:** Specify how long priority levels apply
- **KV cache event API:** Visibility into cache state for upstream applications
- Default eviction follows LRU (Least Recently Used) policy
- Priority-based eviction increases cache hit rate by ~20% [7]

**Text Generation Inference (TGI):**
TGI implements [8]:
- PagedAttention for memory management
- FlashAttention for reduced VRAM usage with padless tensors
- Automatic warmup phase that estimates token budget based on available hardware
- Calculates available VRAM as: `GPU VRAM - Model VRAM - Prefill KV Cache VRAM` [8]

### Q3: What is RunAI and how does it manage GPU resources for inference workloads?

**Overview:**
NVIDIA Run:ai is a GPU orchestration platform built on Kubernetes that provides intelligent scheduling and resource management for AI workloads [9][10]. It consists of two components:
- **Run:ai cluster:** Scheduling and workload management extending Kubernetes
- **Run:ai control plane:** Resource management, workload submission, monitoring [10]

**Key Capabilities for Inference:**

*Dynamic GPU Fractioning:*
- GPUs can be divided into smaller units (e.g., 0.5 GPU allocations) [11]
- Memory isolation enforced at runtime
- Users specify memory requirements; scheduler allocates on-demand
- Supports Request (guaranteed minimum) and Limit (burstable upper bound) [11]

*Intelligent Workload Scheduling:*
- Inference workloads automatically get highest default priority [12]
- Training jobs yield to inference during peak periods
- Topology-aware scheduling for distributed inference
- Gang scheduling for multi-pod workloads [13]

*Autoscaling:*
- Dynamic scaling based on latency, throughput, or concurrency metrics
- Scale-to-zero capability for idle models [12]

**Benchmarking Results:**
- Up to 2x greater user capacity on existing hardware during peak periods [11]
- Three NIM microservices consolidated from 3 dedicated H100s to ~1.5 H100s while retaining 91-100% throughput [12]

### Q4: How do KV cache configurations interact with GPU memory allocation in containerized/Kubernetes environments?

**The Core Challenge:**
KV cache is managed at the **application level** (inference engine), but GPU memory is allocated at the **infrastructure level** (Kubernetes/orchestrator). This creates coordination challenges [14][15].

**Application-Level Configuration (vLLM example):**
- `--gpu-memory-utilization`: Fraction of GPU memory to use (default 0.9) [16]
- `--kv-cache-memory-bytes`: Explicit KV cache size per GPU [16]
- `--block-size`: Size of contiguous cache blocks (1-256 tokens) [16]
- `--enable-prefix-caching`: Reuse KV cache across requests with shared prefixes [16]
- `--kv-cache-dtype`: Quantization (auto, fp8, bfloat16) [16]
- `--cpu-offload-gb`: Amount to offload to CPU memory [16]

**Infrastructure-Level Considerations:**

*Tiered Storage (LMCache on GKE):*
Expands KV cache beyond GPU HBM using tiered storage [14]:
- Tier 1: GPU HBM (fastest)
- Tier 2: CPU RAM
- Tier 3: Local SSD
This dramatically increases total cache size and hit ratio.

*KV Cache-Aware Routing (llm-d):*
Kubernetes-native framework that routes requests to pods with relevant cached content [15]:
- Tracks cache state across vLLM pods
- Prefix-aware scoring routes based on prompt similarity
- Results: 70% reduction in compute time for repeated prompts, 3x more concurrent users [15]

**Key Insight:**
KV cache handling is primarily **application-level** (inference engine configuration), but **GPUaaS-level** orchestration (Run:ai, llm-d) can optimize cache utilization through intelligent routing, memory isolation, and tiered storage integration.

### Summary

KV caching is a fundamental optimization for LLM inference that trades memory for compute, storing intermediate attention computations to avoid redundant calculations. All major inference frameworks (vLLM, TensorRT-LLM, TGI) implement sophisticated KV cache management, with PagedAttention being the dominant approach for eliminating memory fragmentation.

**Key themes:**
- KV cache memory can exceed model weight memory for long contexts, making it the primary bottleneck for scaling inference
- Memory management happens at two levels: application (inference engine) and infrastructure (orchestrator)
- Modern solutions address this through quantization, tiered storage (GPU→CPU→SSD), and intelligent routing

**For the user's purpose (RunAI setup):** KV cache configuration is primarily an **application-level concern** handled by the inference engine (vLLM, TensorRT-LLM). However, RunAI's GPU fractioning and memory isolation features interact with these settings—the orchestrator allocates GPU memory, while the inference engine manages how that memory is used for KV cache. Proper coordination requires understanding both layers.

---

## Deeper Dive

### Subtopic 1: KV Cache Optimization Techniques

#### Q1: What are the practical methods for reducing KV cache memory footprint?

**Quantization:**
KV cache can be quantized from FP16/BF16 to lower precision formats [19][20]:

| Format | Memory Reduction | Accuracy Impact |
|--------|------------------|-----------------|
| FP8 (E4M3) | 50% vs FP16 | Minimal (<1%) |
| NVFP4 | 75% vs FP16, 50% vs FP8 | <1% on benchmarks |
| INT4/INT8 | 75-87.5% vs FP16 | Slight loss with INT4 |

vLLM configuration for FP8 KV cache [19]:
```bash
--kv-cache-dtype fp8
--calculate-kv-scales  # Recommended: calibrate scales
```

NVFP4 (NVIDIA's 4-bit format) provides up to 3x better TTFT by allowing 2x more context to remain on-device [20].

**Offloading:**
When GPU HBM is insufficient, KV cache can be offloaded to [17]:
- CPU RAM (fast retrieval, ~10x slower than HBM)
- Local SSD (larger capacity, higher latency)
- Network storage (cross-instance sharing)

vLLM supports offloading via `--cpu-offload-gb` and integration with LMCache [16].

**Tiered Storage (LMCache):**
Combines multiple storage tiers for optimal cost/performance [14]:
- Tier 1: GPU HBM (fastest, limited)
- Tier 2: CPU RAM (larger, slower)
- Tier 3: Local SSD (largest, slowest)

#### Q2: How do prefix caching and cache sharing work?

**Automatic Prefix Caching (APC):**
vLLM's APC reuses KV cache blocks across requests with shared prefixes [21]:
- Blocks are hashed based on token content
- Matching hashes allow cache reuse without recomputation
- Enabled with `--enable-prefix-caching`

**Use cases where prefix caching excels:**
- System prompts shared across requests
- RAG pipelines with common document chunks
- Multi-turn conversations
- Template-based prompts

**Multi-tenant isolation:**
For security in shared environments, vLLM supports `cache_salt` parameter [21]:
```json
{
  "messages": [...],
  "cache_salt": "tenant-specific-salt"
}
```
This ensures cache sharing only within the same trust group.

**RadixAttention (SGLang):**
Alternative approach using radix tree for prefix matching [3]:
- Keeps KV cache in GPU memory after request completion
- Uses LRU eviction when memory is full
- Cache-aware scheduling prioritizes requests matching cached prefixes

### Subtopic 2: RunAI Integration Patterns

#### Q3: How should KV cache settings be configured with RunAI GPU fractions?

**The coordination challenge:**
When using RunAI GPU fractions, there are two memory limits to consider:
1. **RunAI allocation:** The GPU memory fraction assigned to the workload
2. **Inference engine setting:** `--gpu-memory-utilization` in vLLM

**Configuration guidance:**

*When using full GPU (1.0 fraction):*
- Use default `--gpu-memory-utilization=0.9`
- vLLM will auto-calculate KV cache size

*When using fractional GPU (e.g., 0.5):*
- RunAI enforces memory isolation at runtime [11]
- Set `--gpu-memory-utilization` relative to your fraction
- Example: 0.5 GPU fraction → consider `--gpu-memory-utilization=0.85` (of your allocated portion)
- Or use `--kv-cache-memory-bytes` for explicit control [16]

**Handling preemption warnings:**
If you see KV cache preemption warnings [22]:
1. Increase `--gpu-memory-utilization`
2. Decrease `--max-num-seqs` or `--max-num-batched-tokens`
3. Enable KV cache quantization (`--kv-cache-dtype fp8`)
4. Request larger GPU fraction from RunAI

#### Q4: Best practices for LLM inference with RunAI in airgapped/on-prem environments

**Air-gapped NIM deployment [23]:**
1. Pre-download model and create model store on internet-connected machine
2. Transfer model repository to air-gapped environment
3. Run NIM with `NIM_MODEL_NAME=/model-repo` (local path)
4. Do not set `HF_TOKEN` in air-gapped environment

**RunAI-specific considerations:**
- Use RunAI's self-hosted installation with airgapped package [24]
- Pre-pull container images to local registry
- Configure RunAI to use local registry: `<REGISTRY_URL>`

**Memory planning for on-prem:**
Calculate total memory needed:
```
Total = Model Weights + KV Cache + Overhead
KV Cache = 2 × 2 × n_layers × n_heads × head_dim × max_tokens × batch_size
```

For constrained environments:
- Enable FP8 KV cache quantization
- Use prefix caching for repeated prompts
- Consider CPU offloading if HBM is limited

### Summary

The Deeper Dive research reveals that KV cache optimization operates at multiple levels:

**Application-level optimizations** (configured in inference engine):
- Quantization (FP8, NVFP4) for 50-75% memory reduction
- Prefix caching for shared prompt reuse
- Offloading to CPU/SSD for extended capacity

**Infrastructure-level optimizations** (configured in orchestrator):
- GPU fractioning with memory isolation (RunAI)
- KV cache-aware routing (llm-d)
- Tiered storage integration (LMCache)

**Key insight for the user's purpose:** When setting up RunAI for LLM inference, KV cache handling is primarily an **application-level concern**. Configure the inference engine (vLLM, TensorRT-LLM) with appropriate `--gpu-memory-utilization` and `--kv-cache-dtype` settings. RunAI's role is to allocate and isolate GPU memory—it does not directly manage KV cache. However, RunAI's intelligent scheduling and GPU fractioning features can indirectly optimize cache utilization by enabling multi-model co-location and autoscaling.

