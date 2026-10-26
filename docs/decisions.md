# Decisions

These choices establish Enma's implementation direction. They do not imply that the runtime exists. Unresolved choices belong in [Open implementation questions](PROJECT_INTENT.md#open-implementation-questions).

## Own execution and reuse tensor primitives

Use Candle with CUDA for tensor operations and existing matrix multiplication kernels. Enma owns the model forward pass, generation, scheduling, and KV management so these mechanisms remain inspectable.

A ready-made generation API would hide the runtime being built. Writing a general tensor library would add work outside the project's purpose. Custom kernels are justified when a required execution layout needs them or profiling demonstrates a benefit.

## Establish a simple correctness baseline

Start with greedy single-request execution and contiguous KV storage. Introduce sampling, concurrency, and paging after their prerequisites work.

This provides a small reference path for diagnosing later failures. Some early storage code may be replaced. Keep the baseline useful for comparisons without preserving obsolete public APIs or adding a separate construction path for every milestone.

## Separate scheduling from GPU execution

The scheduler chooses work and the worker executes it. This keeps admission and fairness policy separate from tensor operations and synchronization.

Start with one Rust crate and clear modules. Separate crates or extensible backend interfaces need a concrete benefit; a possible future model is not enough on its own.

## Bound resource use before tuning throughput

Queues, output buffers, active requests, and GPU memory need explicit limits. Measure numeric defaults on the target GPU rather than treating advertised VRAM as available cache capacity.

Reject requests that exceed supported limits, queue temporarily blocked requests within capacity, and reject excess arrivals. Preemption follows reliable admission and resource accounting because it adds recomputation and another request state to manage.

## Share immutable prefix blocks first

Prove allocation and reclamation before adding radix lookup. Initially share completed immutable blocks and keep writable tails private.

This simplifies ownership when requests diverge. Reference tracking still has to protect active and in-flight use; a prefix lookup structure does not solve block lifetime or eviction.

## Keep network compatibility at the boundary

Use a typed internal request and streaming interface, then add a thin adapter for the first client. OpenAI compatibility remains undecided until the integration requires it.

This lets model execution and scheduling develop without depending on a particular wire format. LLMGate is a documentation reference here, not a confirmed integration dependency.

## Keep plans separate from evidence

Project intent owns future requirements and unresolved questions. Architecture describes implemented behavior, manual testing provides runnable procedures, and benchmarks records measurements and their limits.

This prevents a proposed component or planned check from appearing to be implemented or verified. Update the owning document and link to it rather than maintaining repeated explanations.
