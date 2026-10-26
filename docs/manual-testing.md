# Manual testing

There is no runnable implementation yet. No build, model-loading, serving, or test command has been verified. Planned acceptance scenarios live in [Validation requirements](PROJECT_INTENT.md#validation-requirements).

## Setup and configuration

Add prerequisites, pinned tool and model versions, setup commands, and configuration defaults when the first executable path exists. Include how to confirm GPU execution and where model assets are stored. Do not infer CUDA compatibility from the GPU model alone.

## Running checks

For each available behavior, document the exact command, input, expected output or failure, and cleanup. Keep setup here and link to it from benchmarks. Automated test commands also belong here once they exist.

Record the source revision, environment, runtime limits, observed result, and relevant logs when running a check. Mark it passed, failed, blocked, or skipped. A manual run does not replace automated coverage of request transitions, cache ownership, and failure handling.
