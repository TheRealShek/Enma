# Benchmarks

No Enma measurements have been collected. The benchmark harness and performance targets have not been established. Planned metrics and workloads live in [Measurement requirements](PROJECT_INTENT.md#measurement-requirements).

## Recording results

Each result must identify the question, source revision, exact workload, environment, reproduction command, raw-data location, correctness evidence, measurements, and limitations. Link to [Manual testing](manual-testing.md) for general setup instead of repeating it.

Record model and tokenizer revisions, precision, dependency versions, build profile, GPU and VRAM, driver and CUDA versions, runtime limits, competing GPU activity, warmup, repetitions, and sampling settings. If the source is uncommitted, preserve a source fingerprint and identify that explicitly.

Define timing boundaries, sample counts, percentile calculations, arrival patterns, and cache state. Explain whether tokenization and transport are included. GPU execution is asynchronous, so state the synchronization or device timing method used. Keep startup and warmup separate from steady-state results.

Report accepted, rejected, failed, and cancelled work alongside successful requests. Keep historical results labeled with their original configuration. Store referenced plots in `images/` and link raw samples from each report.

No performance improvement is established until correctness checks and a reproducible comparison support it. Single-GPU results cannot establish distributed performance.
