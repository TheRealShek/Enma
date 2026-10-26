# Enma

Enma is an LLM inference engine written in Rust. The project explores model execution, GPU memory management, request scheduling, and efficient concurrent serving.

## Project goals

- Implement the complete generation path for a pretrained language model.
- Schedule concurrent requests to use GPU capacity efficiently.
- Manage KV cache allocation, sharing, eviction, and reclamation.
- Understand performance through correctness checks, benchmarks, and profiling.
- Implement selected GPU kernels and extend the runtime toward distributed serving.

## Initial setup

- Language: Rust.
- GPU: NVIDIA RTX 3060 with 6 GB of VRAM.
- Model: Qwen/Qwen2.5-0.5B-Instruct.
- Precision: FP16 weights, activations, and KV cache, with FP32 intermediate calculations where numerical stability requires them.
- Tensor backend: Candle with CUDA support.

The published BF16 weights will be converted to FP16 during loading. Initial context lengths and concurrency will stay within measured memory limits.

## Phase 1: Correct model execution

Implement the generation path for one request.

- Weight loading, tokenizer integration, and chat template handling.
- Transformer forward pass, including embeddings, RMSNorm, RoPE, grouped-query attention, the MLP, and output projection.
- Causal masking and contiguous KV caching.
- Separate prefill and decode execution.
- Greedy decoding, temperature sampling, top-k, top-p, and stopping conditions.
- Correctness comparisons against a pinned Hugging Face Transformers reference.

This phase establishes a correct baseline before concurrent execution.

## Phase 2: Concurrent serving and memory management

Implement the runtime that manages requests and GPU memory.

- Separate scheduler and GPU worker responsibilities.
- Request queues, state transitions, admission limits, cancellation, and cleanup.
- Continuous batching with token and sequence budgets.
- Paged KV allocation and attention that reads the paged cache.
- Preemption and recomputation under memory pressure.
- Chunked prefill.
- Radix prefix caching with reference tracking and eviction.
- A minimal streaming interface for gateway integration.

This phase focuses on request isolation, memory ownership, and predictable behavior under load.

## Phase 3: GPU optimization and distributed serving

Investigate execution bottlenecks and extend the runtime beyond one GPU.

- Profile CPU scheduling, kernel launches, GPU execution, allocations, and transfers.
- Implement selected custom CUDA kernels called from Rust.
- Evaluate kernel fusion, CUDA graphs, and CPU/GPU overlap.
- Investigate latency and throughput across different workloads.
- Implement tensor parallel execution and compare it with independent replicas.
- Explore replica routing and autoscaling based on queue pressure and latency targets.

Distributed execution requires additional hardware. Single-GPU experiments won't establish distributed performance.

## Implementation boundaries

Enma will own the model forward pass, generation loop, scheduler, request state, and KV cache management.

Pretrained weights, an existing tokenizer library, Candle's tensor operations, and existing matrix multiplication kernels will be reused.

Cache ownership, tensor layouts, allocations, transfers, and synchronization must remain inspectable. Ready-made generation APIs must not hide the runtime components being implemented.

## Design references

- vLLM for scheduling, paged KV management, and scheduler/worker boundaries.
- SGLang for radix prefix caching and cache-aware scheduling.
- NVIDIA TensorRT-LLM for packed execution, in-flight batching, and chunked prefill.

These references guide individual design decisions. Enma doesn't aim to reproduce an entire existing engine.

## Validation intent

Each phase must preserve generation correctness and resource accounting. Performance changes must be evaluated with reproducible workloads and documented hardware limits.

Benchmark results and plots belong in the public README or dedicated benchmark documents.

## Outside the initial scope

- Model training.
- A general-purpose tensor library.
- Multiple model architectures.
- Production feature parity.

Quantization and speculative decoding remain later extensions after the core runtime is correct and measured.
