# KV Caches and RunAI

**Author:** Background Research Agent (BEARY)  
**Date:** 2026-03-02

---

## Abstract

This whitepaper examines Key-Value (KV) caches in the context of on-premises LLM inference deployments using NVIDIA Run:ai. KV caching is a fundamental optimization that trades memory for compute, storing intermediate attention computations to avoid redundant calculations during autoregressive text generation. The paper addresses a critical question for MLOps engineers: **Is KV cache handling an application-level or GPUaaS-level configuration concern?**

The research concludes that KV cache management is primarily an **application-level responsibility**, configured within inference engines like vLLM or TensorRT-LLM. However, Run:ai's GPU orchestration features—including fractional GPU allocation, memory isolation, and intelligent scheduling—interact with these settings and can indirectly optimize cache utilization. Understanding both layers is essential for effective deployment.

---

## Introduction

Large Language Models (LLMs) have become central to enterprise AI initiatives, but their deployment presents significant infrastructure challenges. One of the most critical—and often misunderstood—aspects of LLM inference is the management of the Key-Value (KV) cache.

For organizations deploying LLMs on-premises with GPU orchestration platforms like NVIDIA Run:ai, a fundamental question arises: Where does KV cache configuration belong? Is it something the inference engine handles automatically, or does the GPU orchestration layer need to be aware of it?

This whitepaper addresses three key questions:
1. What are KV caches and why do they matter for inference performance?
2. How do modern inference frameworks manage KV caches?
3. How should KV cache settings be configured when deploying with Run:ai, particularly in airgapped environments?

---

## Background

### What is a KV Cache?

During LLM inference, the model generates text one token at a time through an autoregressive process. Each new token requires computing attention scores against all previous tokens. Without optimization, this leads to redundant O(n²) computation as the sequence grows [1].

The KV cache solves this by storing the Key (K) and Value (V) vectors computed during the attention mechanism. Once computed, these vectors are cached and reused for subsequent token generation, reducing per-step complexity from O(n²) to O(n) [1][2].

**How it works:**
1. During the initial forward pass (prefill), the model computes and stores K/V vectors for all input tokens
2. For each subsequent token, only the new token's K/V vectors are computed
3. Previously computed vectors are retrieved from cache, avoiding recomputation

This optimization provides substantial speedups—approximately 5x even for small models—with benefits increasing for longer sequences and larger models [1].

### The Memory Challenge

The trade-off is memory consumption. KV cache size grows linearly with batch size and sequence length. The formula for per-token KV cache memory (in FP16) is:

```
KV Cache per token = 2 × 2 × n_layers × n_heads × head_dim bytes
```

For practical context [3][4]:
- **Llama-2-7B:** ~0.5 MB per token
- **BLOOM-176B:** ~4 MB per token
- **10,000 token context for Llama-2-7B:** ~5 GB for KV cache alone

In many scenarios, KV cache memory can exceed the memory required for model weights, making it the primary bottleneck for scaling inference [3].

---

## Inference Framework KV Cache Management

Modern inference frameworks implement sophisticated KV cache management strategies. Understanding these is essential for proper configuration.

### vLLM and PagedAttention

vLLM introduced PagedAttention, a memory management technique inspired by operating system virtual memory paging [5][6]. Key innovations include:

- **Non-contiguous block allocation:** KV cache is stored in fixed-size blocks that don't need to be contiguous in memory
- **On-demand allocation:** Blocks are allocated as needed, eliminating internal fragmentation
- **Near-zero waste:** Achieves <4% memory waste compared to 80% in naive implementations [3]

vLLM also supports **Automatic Prefix Caching (APC)**, which reuses KV cache blocks across requests with shared prefixes—particularly valuable for system prompts and RAG pipelines [21].

### TensorRT-LLM

NVIDIA's TensorRT-LLM provides fine-grained KV cache control [7]:

- **Priority-based eviction:** Set retention priorities for specific token ranges (e.g., keep system prompts longer)
- **Duration-based retention:** Specify how long priority levels apply
- **KV cache event API:** Visibility into cache state for upstream applications

Priority-based eviction can increase cache hit rates by approximately 20% [7].

### Text Generation Inference (TGI)

Hugging Face's TGI implements [8]:
- PagedAttention for memory management
- FlashAttention for reduced VRAM usage
- Automatic warmup phase that estimates token budget based on available hardware

---

## KV Cache Optimization Techniques

### Quantization

KV cache can be quantized to reduce memory footprint with minimal accuracy impact [19][20]:

| Format | Memory Reduction | Accuracy Impact |
|--------|------------------|-----------------|
| FP8 (E4M3) | 50% vs FP16 | <1% |
| NVFP4 | 75% vs FP16 | <1% |
| INT4/INT8 | 75-87.5% vs FP16 | Slight loss with INT4 |

**vLLM configuration:**
```bash
vllm serve model-name \
  --kv-cache-dtype fp8 \
  --calculate-kv-scales
```

NVFP4 (NVIDIA's 4-bit format) can provide up to 3x better time-to-first-token (TTFT) by allowing 2x more context to remain on-device [20].

### Offloading

When GPU High Bandwidth Memory (HBM) is insufficient, KV cache can be offloaded to [17]:
- **CPU RAM:** Fast retrieval, ~10x slower than HBM
- **Local SSD:** Larger capacity, higher latency
- **Network storage:** Enables cross-instance sharing

vLLM supports offloading via `--cpu-offload-gb` and integration with LMCache for tiered storage [14][16].

### Prefix Caching

Prefix caching reuses KV cache blocks across requests with shared content [21]. Ideal use cases:
- System prompts shared across requests
- RAG pipelines with common document chunks
- Multi-turn conversations
- Template-based prompts

For multi-tenant environments, vLLM supports cache isolation via `cache_salt` parameter to prevent cross-tenant cache sharing [21].

---

## Run:ai and GPU Orchestration

### Overview

NVIDIA Run:ai is a GPU orchestration platform built on Kubernetes that provides intelligent scheduling and resource management for AI workloads [9][10]. It consists of:

- **Run:ai cluster:** Scheduling and workload management extending Kubernetes
- **Run:ai control plane:** Resource management, workload submission, monitoring

### Key Capabilities for Inference

**Dynamic GPU Fractioning:**
- GPUs can be divided into smaller units (e.g., 0.5 GPU allocations) [11]
- Memory isolation enforced at runtime
- Supports Request (guaranteed minimum) and Limit (burstable upper bound)

**Intelligent Workload Scheduling:**
- Inference workloads automatically receive highest default priority [12]
- Training jobs yield to inference during peak periods
- Topology-aware scheduling for distributed inference
- Gang scheduling for multi-pod workloads [13]

**Autoscaling:**
- Dynamic scaling based on latency, throughput, or concurrency metrics
- Scale-to-zero capability for idle models [12]

### Benchmarking Results

Run:ai benchmarks demonstrate significant efficiency gains [11][12]:
- Up to 2x greater user capacity on existing hardware during peak periods
- Three NIM microservices consolidated from 3 dedicated H100s to ~1.5 H100s while retaining 91-100% throughput

---

## Configuring KV Cache with Run:ai

### The Two-Layer Model

When deploying LLM inference with Run:ai, memory management operates at two levels:

1. **Infrastructure level (Run:ai):** Allocates and isolates GPU memory for workloads
2. **Application level (inference engine):** Manages how allocated memory is used for model weights, KV cache, and overhead

**Run:ai does not directly manage KV cache.** Its role is to allocate GPU memory fractions and enforce isolation. The inference engine then decides how to use that allocated memory.

### Configuration Guidance

**Full GPU allocation (1.0 fraction):**
```bash
vllm serve model-name \
  --gpu-memory-utilization 0.9  # Default, uses 90% of GPU memory
```

**Fractional GPU allocation (e.g., 0.5):**
```bash
vllm serve model-name \
  --gpu-memory-utilization 0.85  # Of your allocated fraction
  # OR
  --kv-cache-memory-bytes 8G     # Explicit size for precise control
```

When using Run:ai GPU fractions, the inference engine sees only the allocated portion. Set `--gpu-memory-utilization` relative to your fraction, or use `--kv-cache-memory-bytes` for explicit control [16].

### Handling Preemption Warnings

If you encounter KV cache preemption warnings [22]:

```
WARNING: Sequence group preempted due to insufficient KV cache space
```

Resolution options:
1. Increase `--gpu-memory-utilization`
2. Decrease `--max-num-seqs` or `--max-num-batched-tokens`
3. Enable KV cache quantization (`--kv-cache-dtype fp8`)
4. Request larger GPU fraction from Run:ai

### Memory Planning Formula

For on-premises deployments, calculate total GPU memory requirements:

```
Total Memory = Model Weights + KV Cache + Overhead (~10-30%)

KV Cache = 2 × 2 × n_layers × n_heads × head_dim × max_tokens × batch_size
```

---

## Air-Gapped Deployment Considerations

For airgapped/on-premises environments [23][24]:

**NVIDIA NIM deployment:**
1. Pre-download model and create model store on internet-connected machine
2. Transfer model repository to air-gapped environment
3. Run NIM with local path: `NIM_MODEL_NAME=/model-repo`
4. Do not set `HF_TOKEN` in air-gapped environment

**Run:ai self-hosted installation:**
1. Use Run:ai's airgapped installation package
2. Pre-pull container images to local registry
3. Configure Run:ai to use local registry

**Memory optimization for constrained environments:**
- Enable FP8 KV cache quantization
- Use prefix caching for repeated prompts
- Consider CPU offloading if HBM is limited

---

## Discussion

### Application vs. Infrastructure Responsibility

The research clearly indicates that **KV cache management is primarily an application-level concern**. The inference engine (vLLM, TensorRT-LLM, TGI) handles:
- KV cache allocation within available GPU memory
- Memory fragmentation through PagedAttention
- Cache eviction policies
- Quantization and offloading

Run:ai's role is complementary:
- Allocating GPU memory fractions to workloads
- Enforcing memory isolation between workloads
- Intelligent scheduling to maximize GPU utilization
- Autoscaling based on demand

### Indirect Optimization Through Orchestration

While Run:ai doesn't directly manage KV cache, its features can indirectly optimize cache utilization:

- **GPU fractioning** enables multi-model co-location, improving overall cluster efficiency
- **Memory isolation** prevents workloads from interfering with each other's KV cache
- **Autoscaling** can spin up additional replicas when KV cache pressure causes latency degradation

### Emerging Patterns

Advanced deployments are beginning to integrate KV cache awareness into the orchestration layer:
- **llm-d** provides KV cache-aware routing at the Kubernetes level [15]
- **LMCache** enables tiered storage across GPU, CPU, and SSD [14]
- **NVIDIA Dynamo** offers KV cache offloading with storage provider integration [17]

These represent a convergence of application and infrastructure concerns, though they remain optional enhancements rather than core requirements.

---

## Conclusion

For MLOps engineers setting up Run:ai for LLM inference, the key takeaways are:

1. **KV cache is an application-level concern.** Configure it in your inference engine (vLLM, TensorRT-LLM), not in Run:ai.

2. **Run:ai allocates memory; the inference engine manages it.** When using GPU fractions, adjust `--gpu-memory-utilization` or use `--kv-cache-memory-bytes` for explicit control.

3. **Optimize KV cache through the inference engine:**
   - Enable FP8 quantization for 50% memory reduction
   - Use prefix caching for shared prompts
   - Consider CPU offloading for memory-constrained environments

4. **Run:ai indirectly supports KV cache efficiency** through GPU fractioning, memory isolation, and intelligent scheduling.

5. **For airgapped deployments,** pre-download models and use local paths; KV cache configuration remains the same.

Understanding both layers—application and infrastructure—is essential for effective LLM deployment. The inference engine handles the complexity of KV cache management; Run:ai ensures efficient GPU resource allocation across your cluster.

---

## References

See `kv-caches-runai-references.md` for the full bibliography.

