---
status: "live"
initial_date: "2026-10-29"
---

# ADR: Establish Parameterized Manual Dispatch as the Initial Way to Deploy and Fallback for Ephemeral Environments

## Context and Problem Statement

The workspace requires on-demand provisioning of isolated, short-lived ephemeral environments (comprising compute containers, isolated state stores, and dynamic DNS/routing bindings) to validate branch-level changes prior to mainline integration. While future architectural iterations will investigate automated, event-driven triggers—such as Git webhook listeners, Pull Request label state machines, or chatops bots—the system currently lacks a foundational invocation mechanism.

Coupling the core provisioning logic directly to external event payloads (e.g., GitHub `pull_request` webhook schemas) creates a brittle architecture. If webhook delivery fails, third-party APIs rate-limit requests, or event schemas mutate, engineers are blocked from deploying test environments. Furthermore, automated triggers cannot be reliably built without first validating the underlying infrastructure-as-code (IaC) state transitions and teardown lifecycles in isolation. The architecture must define how ephemeral environment provisioning is invoked today so that it serves both as the foundational execution primitive ("the prior") for future automated triggers and as the permanent, out-of-band fallback mechanism that outlives transient automation layers.

## Decision Drivers

* **Trigger-to-Provisioner Decoupling:** The environment provisioning engine must accept a strictly typed, canonical input contract (e.g., `git_ref`, `environment_id`, `ttl_seconds`, `resource_profile`) completely agnostic of the upstream caller's transport or event schema.
* **Out-of-Band Fault Tolerance (Fallback Survivability):** Provisioning and teardown must remain executable when event-bus brokers, webhook relays, or SCM integration apps experience outages or payload drops.
* **Compositional Primacy ("The Prior" Requirement):** The mechanism must establish the baseline execution pipeline that future automated mechanisms (PR webhooks, commit-status listeners, schedule-based triggers) will invoke via composition or delegation, eliminating duplicate provisioning code paths.
* **Compute Quota and Cost Containment:** Avoiding unconstrained environment proliferation (e.g., provisioning a full stack on every `git push` event across all branches) before automated garbage-collection and lease-expiration controllers are proven in production.
* **Authentication and Auditability:** Every environment mutation must bind to an authenticated identity token with RBAC enforcement and emit an immutable audit log entry recording the actor, target commit SHA, and parameter overrides.

## Considered Options

* **Option 1: Parameterized Manual CI/CD Workflow Dispatch Wrapping a Canonical CLI/Script Contract**
* **Option 2: Event-Driven SCM Webhook Provisioning Only (Implicit Lifecycle on PR Events)**
* **Option 3: Direct Local Workstation Execution Against Cloud Provider APIs**

## Decision Outcome

Chosen option: **"Option 1: Parameterized Manual CI/CD Workflow Dispatch Wrapping a Canonical CLI/Script Contract"**, because it establishes a deterministic, schema-validated invocation boundary inside a controlled runner environment. By separating the provisioning execution script (the canonical contract) from the manual CI dispatch trigger, it immediately unblocks on-demand deployments, acts as the exact callable substrate ("the prior") that future automated triggers will invoke, and guarantees a permanent, high-availability fallback path when automated event pipelines fail or require bypass.

### Benefits

* Good, because future automated triggers (e.g., PR label webhooks or bot commands) can be implemented as thin adapters that parse an external event and invoke the exact same workflow/script contract, preventing state-drift between manual and automated deployments.
* Good, because when webhook deliveries fail or SCM event queues stall, engineers retain a deterministic, RBAC-governed UI and API entrypoint to force-provision, update, or force-destroy an environment by specifying an explicit Git SHA.
* Good, because infrastructure state locks (e.g., Terraform/OpenTofu state backends or Kubernetes namespace leases) are mutated exclusively from ephemeral, hardened CI runners using short-lived OIDC tokens rather than long-lived developer workstation credentials.

### Trade-offs

* Bad, because until automated triggers are layered on top, engineers incur manual context-switching overhead to trigger deployments via CLI or CI UI and must manually coordinate environment teardown if TTL cron jobs fail.
* Bad, because manual parameter entry introduces the risk of human input errors (e.g., mistyped branch refs or non-existent commit SHAs), requiring an explicit pre-flight validation stage in the pipeline before allocating cloud resources.

### Confirmation

Compliance with this ADR will be enforced and verified through the following automated gates:

1. **Contract Schema Enforcement:** A JSON Schema or TypeScript/Zod validator in the provisioning entrypoint script that rejects invocations missing required canonical arguments (`TARGET_REF`, `ENV_NAMESPACE`, `TTL_HOURS`, `ACTION`).
2. **CI Static Analysis Gate:** A pipeline linting rule (e.g., `actionlint` or custom AST check on workflow definitions) verifying that the manual dispatch workflow (`workflow_dispatch` / manual pipeline trigger) contains zero inline provisioning logic and exclusively delegates execution to the versioned provisioning entrypoint script.
3. **Credential Boundary Verification:** Cloud IAM policy assertions verifying that write permissions for ephemeral namespaces/state backends are restricted exclusively to the CI runner OIDC principal, denying direct mutation from individual user IAM principals.

## Pros and Cons of the Options

### Option 1: Parameterized Manual CI/CD Workflow Dispatch Wrapping a Canonical CLI/Script Contract

Architectural pattern where a repository-hosted, versioned provisioning script encapsulates all IaC/container orchestration calls, exposed via an explicit manual trigger in the CI/CD control plane (e.g., GitHub Actions `workflow_dispatch`, GitLab manual job with input variables) and invocable via authenticated CLI/API calls.

* Good, because it satisfies **Trigger-to-Provisioner Decoupling** by forcing the underlying provisioning script to consume standardized environment variables/flags rather than parsing raw webhook JSON payloads.
* Good, because it satisfies **Compositional Primacy**; subsequent automation options under investigation (such as PR comment ops or label-driven triggers) become pure caller workflows that pass extracted commit SHAs into this exact workflow or underlying script.
* Good, because it satisfies **Out-of-Band Fault Tolerance** by remaining accessible via direct API/CLI invocation even if SCM webhook dispatchers or third-party integration apps are degraded.
* Good, because it satisfies **Compute Quota and Cost Containment** by ensuring zero environments are allocated unless an authenticated actor explicitly requests one, establishing a safe baseline while TTL reaper jobs are hardened.
* Neutral, because state management relies on the CI/CD platform's concurrency groups (e.g., `concurrency: ephemeral-${{ inputs.env_id }}`) to serialize conflicting manual runs against the same target environment.
* Bad, because it requires maintaining an explicit input-validation pre-flight step to resolve symbolic Git refs (`branch-name` -> `40-char SHA`) and verify artifact existence before provisioning begins.

### Option 2: Event-Driven SCM Webhook Provisioning Only (Implicit Lifecycle on PR Events)

Architectural pattern where ephemeral environments are created, updated, and destroyed exclusively as a side effect of SCM lifecycle events (`pull_request.opened`, `pull_request.synchronize`, `pull_request.closed`) with no decoupled manual invocation contract.

* Good, because it eliminates human invocation latency during standard PR workflows by binding environment lifecycle directly to branch state transitions.
* Bad, because it violates **Trigger-to-Provisioner Decoupling** by tightly binding the provisioning pipeline to the availability and payload structure of SCM pull request events, preventing deployment of arbitrary commits, tags, or un-PR'd branches.
* Bad, because it violates **Out-of-Band Fault Tolerance**; if a webhook payload is dropped during an SCM outage or a teardown event fails on `pull_request.closed`, there is no independent mechanism to re-trigger provisioning or force-reap orphaned cloud resources.
* Bad, because it violates **Compute Quota and Cost Containment** by triggering full infrastructure convergence on every `git push` to an open PR, multiplying cloud compute allocation and container registry egress for trivial or work-in-progress commits.

### Option 3: Direct Local Workstation Execution Against Cloud Provider APIs

Architectural pattern where engineers execute provisioning scripts (Terraform, Helm, Pulumi, or custom SDK scripts) directly from their local terminal using local credentials to spin up remote ephemeral environments.

* Good, because it satisfies **Out-of-Band Fault Tolerance** at the extreme by removing the CI/CD control plane from the execution path entirely.
* Neutral, because it shares the same underlying CLI script primitive as Option 1, meaning the core script logic is reusable if migrated into CI later.
* Bad, because it violates **Authentication and Auditability** by requiring individual developer workstations to hold broad IAM privileges capable of provisioning compute, networking, and IAM sub-resources, bypassing centralized CI audit logs.
* Bad, because it violates **Compositional Primacy**; local workstation execution cannot serve as a callable runtime target for future automated webhook or schedule-driven triggers without re-architecting the execution environment into a remote runner anyway.
* Bad, because local network drops or SIGINT interruptions during execution leave remote IaC state locks dangling and cloud resources partially provisioned without an automated runner supervisor to handle cleanup.
