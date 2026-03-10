# KV Caches and RunAI — Research Questions

<!-- Research questions for the kv-caches-runai topic. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->

## General Understanding

### Q1: What are KV caches in the context of LLM inference, and why are they important for performance?

**Search terms:**
- KV cache LLM inference explained
- key value cache transformer memory optimization
- KV cache GPU memory management inference

### Q2: How do inference serving frameworks (vLLM, TensorRT-LLM, TGI) implement and manage KV caches?

**Search terms:**
- vLLM KV cache implementation PagedAttention
- TensorRT-LLM KV cache configuration
- text generation inference KV cache management

### Q3: What is RunAI and how does it manage GPU resources for inference workloads?

**Search terms:**
- RunAI GPU orchestration inference workloads
- RunAI Kubernetes GPU scheduling LLM
- RunAI fractional GPU inference deployment

### Q4: How do KV cache configurations interact with GPU memory allocation in containerized/Kubernetes environments?

**Search terms:**
- KV cache GPU memory Kubernetes pods
- LLM inference memory limits container orchestration
- GPU memory fragmentation KV cache containers

---

## Deeper Dive

Based on General Understanding, the topic naturally divides into two subtopics:
1. **KV Cache Optimization Techniques** - Practical methods for reducing memory footprint and improving performance
2. **RunAI Integration Patterns** - How to configure KV cache settings when deploying with RunAI

### Subtopic 1: KV Cache Optimization Techniques

#### Q1: What are the practical methods for reducing KV cache memory footprint (quantization, offloading, compression)?

**Search terms:**
- KV cache quantization FP8 INT4 LLM inference
- KV cache offloading CPU memory SSD vLLM
- KV cache compression techniques production LLM

#### Q2: How do prefix caching and cache sharing work across requests, and when should they be enabled?

**Search terms:**
- vLLM prefix caching configuration multi-tenant
- KV cache sharing system prompt reuse
- RadixAttention SGLang prefix matching

### Subtopic 2: RunAI Integration Patterns

#### Q3: How should vLLM/TensorRT-LLM KV cache settings be configured when using RunAI GPU fractions?

**Search terms:**
- vLLM gpu-memory-utilization RunAI fraction configuration
- TensorRT-LLM KV cache RunAI deployment
- inference engine memory settings Kubernetes GPU limits

#### Q4: What are the best practices for deploying LLM inference with RunAI in airgapped/on-prem environments?

**Search terms:**
- RunAI airgapped deployment LLM inference
- on-premises GPU orchestration LLM serving best practices
- RunAI NIM deployment self-hosted configuration

---

## Redundant Questions

<!-- Move any redundant questions here during review. -->

