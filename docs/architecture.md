# Proposed architecture

**Design stage only.** None of the components or behaviours described here is implemented. The design targets a local portfolio demonstration, without production readiness or performance claims.

## Responsibility boundaries

Developers or GitOps declare an `InferenceService`. The Go/Kubebuilder operator watches it and its owned Deployment and Service, computes desired Deployment and Service state, applies necessary changes and reports status. Kubernetes manages pod scheduling, Deployment rollouts and garbage collection.

GitOps owns the custom resource specification. The operator owns its status and the Deployment and Service fields it generates. GitOps should not also manage the Deployment and Service. The workload loads a packaged model and serves the fixed HTTP contract; the operator does not run inference itself.

## Initial API and serving contract

The proposed namespaced API is `inference.platform.example.com/v1alpha1`, kind `InferenceService`. Its initial spec contains a required image, a replica count defaulting to one, and CPU/memory requests and limits. Validation will reject an empty image, negative replica counts and invalid resource quantities or request/limit relationships. Zero replicas will be allowed as an intentional scale-to-zero state, without autoscaling.

One CPU Python/FastAPI workload will implement the following contract on port 8000:

| Endpoint | Planned meaning |
| --- | --- |
| `GET /livez` | The server is responsive; model loading alone must not cause a liveness failure. |
| `GET /readyz` | Return success only after model initialization and a successful local inference check; return failure while loading or unable to serve. |
| `POST /predict` | Accept `{"text": "example input"}` and return `{"label": "example-label"}`; exact label vocabulary will be documented with the workload. |

The readiness handler will reflect the current ability to perform inference, including initialization failures. Tests will verify a successful prediction when readiness succeeds. Prediction requests before readiness will receive a service-unavailable response if they reach a pod directly. Pod readiness probes will use `/readyz`; liveness probes will use `/livez`. Loading time and probe settings will be established with the example workload, without promising a particular startup time.

The initial model will be bundled into the image for a reproducible CPU demo. Other model formats, serving contracts, remote model stores and GPU scheduling are outside the initial scope.

## Managed resources and reconciliation

Each custom resource will have one deterministically named Deployment and ClusterIP Service in its namespace. A stable per-resource selector will connect the Service to the Deployment's pods. The Service will expose HTTP port 80 to container port 8000. The pod template will carry the image, resource requirements and fixed serving probes; replicas will be set on the Deployment.

Each reconciliation will read current state, derive desired state, create missing managed resources and patch managed fields only when they differ. Repeated reconciliation of unchanged inputs must converge without repeated writes. Watches on owned resources will trigger recovery after deletion or drift. Transient errors will be retried using controller-runtime's reconciliation mechanisms.

Existing objects with conflicting ownership will produce an explicit failure condition; the operator will not adopt or overwrite unrelated objects. Managed metadata and selectors must be designed to prevent collisions, while preserving unrelated metadata where possible.

Image, replica and resource changes will update the Deployment. Kubernetes will perform the rollout. The operator will observe the latest generation and rollout result; it will not automatically restore an earlier spec after a failure.

## Status and failure reporting

The proposed status includes `observedGeneration`, desired/updated/ready/available replica counts, and Kubernetes-style conditions with status, reason, message, observed generation and transition time:

| Condition | Planned meaning |
| --- | --- |
| `Ready` | The latest desired state has converged and all desired replicas are updated and available with model readiness satisfied. False for zero desired replicas, with reason `ScaledToZero`. |
| `Progressing` | The current desired state is being applied or its rollout is still advancing. |
| `Degraded` | A reconciliation error, ownership conflict or Deployment rollout failure prevents convergence. |

Ready will require the Deployment to have observed its current generation and completed the desired rollout; available pods from an earlier image alone will not make a new generation ready. Status must not present stale conditions as results for the current spec. Condition updates will avoid changing transition timestamps when condition status has not changed.

The Deployment will have an explicit progress deadline. A `ProgressDeadlineExceeded` condition will be reflected as a rollout failure. Image pull errors and permanently failing readiness may first appear as progress and later as deadline failure. Messages will describe observed evidence without claiming a root cause the operator cannot establish. Correcting the image or workload must allow conditions to recover. Exact reason names and deadline settings will be finalized and tested during implementation.

## Ownership, deletion and permissions

Both managed resources will have controller owner references to the `InferenceService`. Kubernetes garbage collection will remove them after the InferenceService is deleted. No finalizer is planned because the initial design has no external resources to clean up. Deleting a managed Deployment or Service while its InferenceService exists will cause recreation.

The initial installation will target a configured namespace, with a Role and RoleBinding for custom resources, status, Deployments, Services, events and any leader-election Lease used by the scaffold. CRD installation requires cluster-level access; runtime resource management should not require cluster-wide write permissions. Generated RBAC will be reviewed against the actual controller operations. Broader multi-namespace operation is deferred.

## Constraints and verification

The project excludes billing, a UI, automatic rollback and unrelated infrastructure. The Service is internal to the cluster; external ingress is outside the first version.

Unit tests will cover pure decisions. envtest will cover CRD validation, managed resource reconciliation and status inputs against an API server. kind tests will verify real controllers, model readiness, successful inference, failed rollouts, managed resource recreation and garbage collection. Success criteria concern observable behaviour, not measured throughput or production suitability.
