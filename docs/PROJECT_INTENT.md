# Enma

Enma is an LLM inference engine written in Rust. It will be a usable single-GPU engine built for understanding and measurement, initially for personal use, with public correctness checks and benchmarks.

The main engineering goal is correct, inspectable concurrent serving: run a pretrained model, schedule requests, and manage GPU memory within measured limits.

## Current checkpoint

The repository contains planning documents only. No model execution, serving runtime, or benchmark harness exists yet. The next milestone is a correct single-request forward pass, followed by cached greedy generation.

## Initial stack

- Language: Rust.
- GPU: NVIDIA RTX 3060 with 6 GB of VRAM.
- Model: Qwen/Qwen2.5-0.5B-Instruct.
- Tensor backend: Candle with CUDA support.
- Precision: FP16 weights, activations, and KV cache, with FP32 intermediate calculations where numerical stability requires them.

Convert the published BF16 weights to FP16 during loading. Pin the model revision, tokenizer assets, dependencies, and reference implementation before establishing correctness results. Set context and concurrency limits from measurements on the target GPU.

## First-release scope

The first substantial release will:

- Execute the model and generate responses correctly.
- Stream output for concurrent requests on one GPU.
- Bound queues, execution work, output buffering, and KV cache use.
- Handle cancellation, overload, and cleanup predictably.
- Provide reproducible correctness checks and latency, throughput, and memory benchmarks.

Milestones 1 through 4 define this release. Prefix reuse and further optimization follow once the core runtime is correct and measured. Earlier milestones should remain usable on their own.

## Implementation boundaries

Enma owns the model forward pass, generation loop, sampling, scheduler, request lifecycle, and KV cache management.

Reuse pretrained weights, an existing tokenizer library, Candle tensor operations, and existing matrix multiplication kernels. Cache ownership, tensor layouts, allocations, transfers, and synchronization must remain inspectable. Ready-made generation APIs must not hide the components being implemented.

## Target architecture

This is the planned design across milestones. [Architecture](architecture.md) will describe the implemented path as code lands.

### Components

Start with one Rust crate and separate modules. Introduce additional crates only when a concrete reuse or build boundary justifies them.

- Model: load weights and execute transformer layers, positions, attention, and output projection.
- Generation: select tokens and apply stopping conditions.
- Request runtime: own request state, queues, cancellation, and terminal outcomes.
- Scheduler: select requests and token work for each execution step within budgets.
- GPU worker: execute scheduled work and return results.
- KV cache manager: track block allocation, ownership, references, and reclamation.
- Transport adapter: translate client requests and stream events through the internal interface.

The model has no network responsibilities. The scheduler decides what runs without performing tensor operations. The GPU worker executes the selected work without deciding admission policy.

### Request flow

1. Validate input and generation settings, tokenize, and check supported limits.
2. Admit the request to a bounded queue or return a clear error.
3. Select eligible work and reserve the required cache capacity.
4. Prefill the prompt, then decode new tokens using its cached attention state.
5. Emit output through a bounded stream and check stopping or cancellation.
6. Report a terminal outcome and release request-owned resources once GPU use has finished.

Prefill processes prompt tokens. Decode extends the sequence with generated tokens. Continuous batching allows the participating requests to change between execution steps. Chunked prefill splits a long prompt across steps so other eligible work can progress.

### Request lifecycle

Represent queued, prefill, decode, suspended, and terminal states explicitly. Terminal outcomes distinguish completion, cancellation, and failure. Completion records a reason such as end-of-sequence, a configured stop condition, or an output limit.

Cancellation must work while queued, running, or suspended. It can prevent future work immediately, but memory used by an in-flight GPU operation cannot be reclaimed until that operation is safe to release. Repeated cancellation and cleanup must not double-release resources.

A client disconnect should cancel its request. Define a bounded policy for slow readers before shipping streaming, including when a stalled stream is terminated. Do not block the shared execution loop indefinitely on a client.

### Scheduling and admission

Configure limits for input length, generated tokens, active sequences, queued requests, tokens per execution step, and KV capacity. Account for weights, execution temporaries, cache storage, and other GPU consumers when choosing defaults.

Reject work that cannot fit the supported request limits. Queue temporarily blocked work while queue capacity remains, then reject excess arrivals. Begin with a simple admission order and document how later scheduling policies prevent starvation.

Preemption suspends a request and releases reconstructible KV state. Retain enough token and position state to recompute that state without resampling already generated output. Bound retries and define a terminal failure when progress cannot be made.

### KV cache evolution

Begin with contiguous storage as a correctness baseline. Introduce paging behind a clear cache boundary once basic concurrency works.

Each request maps its logical token positions to physical KV blocks. The manager tracks free capacity and live ownership; attention consumes the corresponding layout and valid lengths. Allocation alone does not provide paged attention.

A gather into contiguous storage may help validate paging, but its copies and temporary memory must be measured. The intended paged runtime reads the paged layout directly. Establish the Candle/CUDA integration path in an early feasibility experiment; a required kernel may precede the later optimization milestone.

For prefix reuse, share immutable completed blocks and keep writable tails private. Track active references separately from retained reusable entries. Evict only blocks that are not needed by active requests or in-flight execution. Define cache compatibility from exact token prefixes and the model/execution configuration that produced their KV state.

### Failure handling and observability

Validate inputs before admission and attach errors and lifecycle events to request IDs. Expose queue depth, active requests, cache occupancy, allocation failures, cancellations, preemptions, and execution timing.

Distinguish request failures from worker failures. If a worker cannot continue safely, stop admitting work and terminate affected streams explicitly rather than leaving them waiting. The recovery policy must be established before the serving milestone is considered complete.

## Milestones

### 1. Correct single-request inference

First reproduce a forward pass against a pinned Hugging Face Transformers reference. Implement embeddings, RMSNorm, RoPE, grouped-query attention, causal masking, the MLP, and output projection.

Then implement separate prefill and decode execution with a contiguous KV cache. Add a CLI, tokenizer integration, chat formatting, greedy decoding, and stopping conditions. Add temperature, top-k, top-p, and seeded sampling after greedy execution is validated.

Done when reference comparisons pass within documented numerical tolerances, cached execution agrees with full-sequence recomputation, and the CLI generates and stops correctly. Sampling must have focused correctness checks; cross-backend sampled text need not match exactly.

### 2. Controlled serving

Introduce a typed request and streaming event interface. Initially queue multiple requests and execute one at a time. Add admission limits, request states, cancellation, bounded output buffering, and cleanup. Add one thin network adapter after identifying its first client.

Done when accepted requests terminate with an explicit outcome, invalid and excess requests receive clear errors, and cancellation or client disconnection releases resources safely. A slow reader must not create unbounded buffering or indefinitely retain request resources.

### 3. Concurrent execution

Separate scheduler decisions from GPU execution. Start with fixed batches and a simple cache layout, then introduce continuous batching with token and sequence budgets. Requests may enter or leave between execution steps.

Done when batched execution preserves per-request correctness, completed slots can be reused, and mixed request lengths make progress within configured memory limits. Measure latency and aggregate throughput against the sequential baseline; document tradeoffs rather than assuming batching improves every workload.

### 4. Paged runtime

Implement fixed-size KV blocks and attention that reads the paged cache. Add chunked prefill, then preemption and recomputation under memory pressure. Run a cache-layout feasibility experiment after milestone 1 so backend limitations are discovered early.

Done when paged execution agrees with the contiguous baseline, allocation and reclamation remain correct across request lifecycles, and memory-pressure workloads complete or fail explicitly without deadlock, leaked blocks, or unbounded retry. Verify preemption, cancellation, and resumption together.

### 5. Prefix reuse and optimization

Add radix prefix lookup, reference tracking, and eviction. Share immutable completed blocks first; keep blocks being extended private. Profile before choosing custom kernels, fusion, CUDA graphs, or CPU/GPU overlap.

Done for each change when correctness and resource accounting remain valid and reproducible measurements explain its benefit, overhead, and workload limits. An optimization without a demonstrated benefit should not become the default.

## Open implementation questions

### Paged attention integration

After single-request correctness, test how the selected Candle/CUDA versions support writing and consuming the proposed cache layout. Compare the contiguous baseline, a gather-based validation path if needed, and direct paged attention.

Record supported operations, temporary storage, correctness results, and required kernel work before committing to the implementation. This is a feasibility gate for the paged runtime, not a reason to delay the initial CLI.

### First client and network protocol

Choose the network contract when the first gateway or application is identified. Keep transport separate from the internal request interface. Decide whether OpenAI compatibility provides concrete integration value before adding it.

### Capacity and scheduling defaults

Measure model residency, execution peaks, and KV growth on the target GPU. Use mixed request lengths to choose token budgets, sequence limits, queue capacity, and an admission policy. Document fairness behavior and overload outcomes.

### Cache layout and block size

Choose layouts and block size using correctness, memory waste, addressing cost, and attention implementation constraints. Define compatibility keys before enabling prefix sharing.

### Numerical tolerances and reproducibility

Pin reference precision and execution settings. Establish absolute and relative tolerances from a justified numerical comparison. Record greedy token agreement separately from logits; close scores can change token selection. Define seeded sampling behavior within Enma without promising cross-backend random-stream equivalence.

### Slow clients and worker failures

Choose output buffer limits, stalled-reader behavior, and the worker recovery policy before completing controlled serving. Verify that affected requests terminate and resources remain accounted for.

## Validation requirements

These are acceptance scenarios for future implementation. Add executable procedures to [Manual testing](manual-testing.md) as each behavior becomes available. Automate deterministic invariants and failure cases where practical.

### Single-request execution

- Compare logits for fixed token sequences against the pinned Transformers reference, using documented tolerances.
- Compare cached decode against full-sequence recomputation at multiple positions and prompt lengths.
- Verify tokenizer and chat-template handling, special tokens, and the documented empty-input behavior.
- Exercise greedy generation, end-of-sequence termination, output limits, and supported stop conditions, including boundaries between streamed pieces.
- Check sampling settings, invalid parameter rejection, and repeatability under the documented seed policy.
- Verify missing, malformed, or incompatible model assets produce actionable failures.

### Controlled serving

- Submit valid requests and verify ordered output and one logical terminal outcome per accepted request.
- Submit invalid and oversized requests and verify rejection before GPU execution.
- Fill the waiting queue and verify excess work receives an explicit overload response.
- Cancel queued and running requests, repeat cancellation, and disconnect clients.
- Stop reading output and verify the configured buffer and stalled-reader policy.
- Verify a subsequent request succeeds after request-level failures and resources return to their expected baseline.
- Exercise the worker-failure path and confirm affected streams terminate under the documented recovery policy.

### Concurrent execution

- Compare each request's execution with its isolated baseline, controlling sampling settings and numerical tolerances.
- Mix prompt and output lengths; confirm one request's tokens, positions, and cache cannot affect another's.
- Complete and cancel requests while admitting new ones; verify slots can be reused safely.
- Send short requests during long prefill and observe progress under the configured scheduling policy.
- Verify queue, token, active-sequence, and memory limits under sustained arrivals.

### Paged cache and memory pressure

- Compare paged attention with contiguous attention, including lengths around block boundaries and partially filled blocks.
- Exercise allocation, release, reuse, exhaustion, and repeated cleanup.
- Cancel requests while GPU work is in flight and confirm referenced storage is not reused early.
- Force preemption and recomputation; compare continued output with uninterrupted execution under controlled generation settings.
- Cancel suspended requests and verify ownership is released.
- Verify capacity exhaustion cannot cause deadlock or unbounded preemption/retry.

### Shared prefixes

- Compare cold-cache and reused-prefix results for identical prefixes and diverging suffixes.
- Finish or cancel one sharing request while others continue; verify their blocks remain valid.
- Evict retained entries while requests are active; verify live and in-flight references remain protected.
- Verify incompatible model/execution configurations cannot reuse cache entries.
- Check that free, active, and retained block accounting reconciles after repeated reuse and eviction.

## Measurement requirements

Measure time to first token, time between output tokens, end-to-end latency, queue time, aggregate output throughput, and peak GPU memory. Report sample counts and latency distributions, including tails.

Use short requests, long prompts, concurrent requests, mixed lengths with staggered arrivals, repeated prefixes, overload, cancellation, and slow readers. Compare changes against the previous Enma baseline under equivalent conditions. Include rejected and failed work in the report.

Distinguish prefill from decode, cold from warm cache, process allocations from cache occupancy, and device-wide memory from engine memory. Profile CPU scheduling, kernel launches, GPU execution, allocations, and transfers before choosing optimizations.

Publish actual results, reproduction commands, raw-data references, and limitations in [Benchmarks](benchmarks.md).

## Later extensions

Distributed execution needs additional hardware and a separate design. Investigate tensor parallelism, independent replicas, routing, and autoscaling only when they can be validated on suitable hardware. Single-GPU results do not establish distributed performance.

Quantization and speculative decoding follow a correct, measured core runtime.

## Outside the initial scope

- Model training.
- A general-purpose tensor library.
- Multiple model architectures.
- Production feature parity with existing engines.
- Distributed serving in the first release.

## Design references

Use vLLM as a reference for scheduling, paged KV management, and scheduler/worker boundaries; SGLang for radix prefix caching; and TensorRT-LLM for packed execution, in-flight batching, and chunked prefill. These inform individual decisions. Enma does not aim to reproduce an entire existing engine.

## Documentation

- [Architecture](architecture.md): implemented execution flow, component boundaries, and ownership.
- [Decisions](decisions.md): agreed non-obvious choices and their tradeoffs.
- [Benchmarks](benchmarks.md): measured results, reproduction details, and limitations.
- [Manual testing](manual-testing.md): runnable setup, configuration, checks, and expected results as they become available.
- `images/`: diagrams and benchmark plots referenced by these documents.

Keep each explanation in its owning document and link to it elsewhere. Repository workflow and documentation responsibilities are defined in [AGENTS.md](../AGENTS.md).
