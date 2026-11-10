# Implementation roadmap

**Current status: design stage; implementation has not started.** All milestones below are planned. Complete one milestone at a time and update this document with evidence as work lands.

## Milestone 1 — Scaffold and define the API

Initialize the Go module and Kubebuilder project. Choose and record compatible tool versions. Define the namespaced `InferenceService` API, schema defaults and validation for image, replicas and CPU/memory resources. Review generated RBAC for the namespace-scoped runtime design.

Acceptance: generated manifests match the API, code builds, schema validation and defaulting have meaningful tests, and documented scaffolding/development commands work from a clean checkout. No workload readiness or reconciliation support is claimed yet.

## Milestone 2 — Implement the serving contract

Create a small Python/FastAPI CPU inference workload with a packaged model, `/predict`, `/livez` and `/readyz`. Document the actual prediction schema and labels, model provenance and applicable licensing. Build a container suitable for local kind use.

Acceptance: tests cover successful prediction, invalid input, loading state and initialization failure. Readiness succeeds only when inference works. Document a verified local image build and workload check.

## Milestone 3 — Reconcile owned resources

Implement idempotent Deployment and Service reconciliation, stable selectors, model probes, controller owner references and watches. Support image, replica and resource changes. Report conflicting ownership without overwriting unrelated objects.

Acceptance: tests verify initial creation, convergence without unnecessary writes, drift correction, updates, child recreation and ownership conflicts. A kind check verifies the Service reaches the workload and parent deletion removes children. Document working install and cleanup commands.

## Milestone 4 — Report operational status

Implement generation-aware replica reporting and `Ready`, `Progressing` and `Degraded` conditions. Define progress deadlines and stable reason names. Handle scale-to-zero explicitly and allow recovery after corrected failures.

Acceptance: tests cover successful rollout, an invalid image, a model that never becomes ready, reconciliation errors, updates during rollout and recovery. Old ready replicas must not falsely signal completion of a new image rollout. Validate real rollout and probe behaviour in kind.

## Milestone 5 — Make the demonstration reproducible

Finish a documented kind workflow and automated end-to-end suite. Demonstrate creation, inference, image/replica/resource updates, child deletion and recovery, rollout failure and correction, and parent deletion with garbage collection. Add CI checks appropriate to the working implementation.

Acceptance: a clean-checkout walkthrough succeeds using documented versions and commands; failure tests have bounded waits and useful diagnostics; documentation identifies remaining limitations and links to verification evidence. The project remains a portfolio demonstration without production or performance claims.

## Testing principles across milestones

- Assert externally visible API, resource and workload behaviour, including failure and recovery paths.
- Use unit tests for isolated decisions and envtest for API/controller integration. Do not treat envtest as proof that pods run or garbage collection completes.
- Use kind for scheduling, Deployment rollouts, Service routing, probes and garbage collection.
- Introduce dependencies only when needed by the current milestone.
- Publish setup commands only after the corresponding scaffold or implementation exists and has been checked.
