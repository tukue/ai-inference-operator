# Repository guidance for Codex

## Current state

This repository is at the design stage. Implementation has not started. Read `README.md`, `docs/architecture.md` and `docs/roadmap.md` before making changes. Their proposed API and behaviour describe intent, not existing functionality.

## Working approach

- Implement one roadmap milestone at a time. Finish its acceptance criteria before starting another, unless the user explicitly changes scope.
- Explain important design decisions, including API semantics, resource ownership, readiness, status and permission boundaries.
- Test observable behaviour and failure cases. Include recovery after correcting failures; avoid tests that only mirror implementation details.
- Keep documentation aligned with working code. Distinguish implemented, verified features from planned features and unverified assumptions.
- Avoid unrelated infrastructure and unnecessary dependencies. Choose the smallest change that serves the active milestone.
- Do not invent working setup commands. Add commands after scaffolding or implementation exists and verify them where possible; state any verification limits.

## Design constraints

- Use Go and Kubebuilder for the operator, kind for local development, and a small Python/FastAPI example workload.
- Start with CPU workloads and the single documented HTTP serving contract.
- Model readiness means the model can serve requests, not simply that the process has started.
- Keep reconciliation idempotent, recover deleted owned children, and report current-generation readiness and rollout failures.
- GitOps manages the custom resource; the operator manages its Deployment and Service. Do not introduce competing ownership.
- Use controller owner references for child cleanup. Add a finalizer only if a concrete cleanup requirement justifies it.
- Use namespace-scoped runtime permissions where practical and review generated RBAC.
- Do not add GPU scheduling, billing, a UI or automatic rollback.
- Do not claim production readiness or measured performance without evidence; neither is a goal of the initial roadmap.

## Verification and reporting

Use checks appropriate to the change. Documentation-only changes need consistency and link review, not a fabricated build. As code arrives, use unit tests, envtest and kind according to their actual capabilities. envtest does not run workload controllers or prove serving readiness.

When finishing work, summarize what changed, what was verified, any material limitations and the next milestone step. Never describe planned functionality as complete.
