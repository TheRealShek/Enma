# Architecture

No runtime has been implemented yet. The [target architecture](PROJECT_INTENT.md#target-architecture) describes the planned components, request flow, and ownership boundaries.

As code lands, this document will explain one request through the implemented system: its entry point, component responsibilities, shared and request-local state, GPU execution, failure paths, and cleanup. Add a short code reading order with links to real entry points.

Keep future components and requirements in project intent. Describe behavior here only when it is supported by code, and distinguish tested behavior from behavior only inspected.
