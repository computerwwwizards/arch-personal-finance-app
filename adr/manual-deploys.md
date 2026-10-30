---
status: "live"
initial_date: "2026-10-29"
---

# ADR: Manual Pipeline Dispatch for Ephemeral Environments as a Permanent Fallback and Baseline Mechanism

## Context and Problem Statement

The workspace needs a way to spin up isolated, short-lived environments on demand to test branch changes before merging them. These environments include compute containers, isolated state stores, and dynamic routing bindings. Future work will explore automated triggers, such as Git webhook listeners or pull request label state machines, but the system currently lacks a baseline invocation method.

Tying the core provisioning logic directly to external event payloads makes the architecture brittle. If webhook delivery fails, application programming interface requests are rate-limited, or event schemas change, engineers cannot deploy test environments. Furthermore, automated triggers cannot be built reliably without first validating infrastructure-as-code state transitions and teardown lifecycles in isolation. The architecture must define how environment provisioning is invoked today so it can serve as the initial execution baseline for future automation and remain as a permanent, out-of-band fallback mechanism that outlives transient automation layers.

## Decision Drivers

* **Trigger-to-Provisioner Decoupling:** The provisioning engine must accept a strictly typed input contract—such as a Git reference, environment identifier, time-to-live duration, and resource profile—that is agnostic of the upstream caller's transport or event schema.
* **Out-of-Band Fault Tolerance:** Provisioning and teardown must remain executable when event brokers, webhook relays, or source control management integration apps experience outages.
* **Baseline Execution Primacy:** The mechanism must establish the foundational pipeline that future automated methods will invoke via composition or delegation, avoiding duplicate provisioning logic.
* **Compute Quota and Cost Containment:** Unconstrained environment proliferation must be prevented—such as provisioning a full stack on every Git push across all branches—before automated garbage collection and lease expiration controllers are proven.
* **Authentication and Auditability:** Every environment mutation must bind to an authenticated identity token with role-based access control and emit an immutable audit log entry recording the actor, target commit hash, and parameter overrides.

## Considered Options

* **Option 1: Parameterized Manual Pipeline Dispatch Wrapping a Shared Script Contract**
* **Option 2: Event-Driven Source Control Management Webhook Provisioning Only**
* **Option 3: Direct Local Workstation Execution Against Cloud Provider Application Programming Interfaces**

## Decision Outcome

Chosen option: **"Option 1: Parameterized Manual Pipeline Dispatch Wrapping a Shared Script Contract,"** because it establishes a deterministic, schema-validated execution boundary inside a controlled runner environment. By separating the provisioning script from the manual pipeline trigger, it immediately unblocks on-demand deployments, acts as the exact callable target for future automated triggers, and guarantees a permanent, high-availability fallback path when automated event pipelines fail.

### Benefits

* Good, because future automated triggers can be implemented as thin adapters that parse an external event and call the exact same script contract, preventing state drift between manual and automated deployments.
* Good, because when webhook deliveries fail or source control management event queues stall, engineers retain a deterministic, role-based access control-governed user interface and application programming interface entry point to force-provision, update, or destroy an environment by specifying an explicit Git commit hash.
* Good, because infrastructure state locks are mutated exclusively from ephemeral, hardened continuous integration runners using short-lived OpenID Connect tokens rather than long-lived developer workstation credentials.

### Trade-offs

* Bad, because until automated triggers are layered on top, engineers incur manual context-switching overhead to trigger deployments via command line interface or user interface and must manually coordinate environment teardown if time-to-live cron jobs fail.
* Bad, because manual parameter entry introduces the risk of human input errors, requiring an explicit pre-flight validation stage in the pipeline before allocating cloud resources.

### Confirmation

Compliance with this architecture decision record will be enforced and verified through the following automated gates:

1. **Contract Schema Enforcement:** A validation script in the entry point that rejects invocations missing required arguments (`TARGET_REF`, `ENV_NAMESPACE`, `TTL_HOURS`, `ACTION`).
2. **Continuous Integration Static Analysis Gate:** A pipeline linting rule verifying that the manual dispatch workflow contains zero inline provisioning logic and exclusively delegates execution to the versioned script.
3. **Credential Boundary Verification:** Cloud access policy assertions verifying that write permissions for ephemeral namespaces and state backends are restricted exclusively to the runner principal, denying direct mutation from individual user identities.

## Pros and Cons of the Options

### Option 1: Parameterized Manual Pipeline Dispatch Wrapping a Shared Script Contract

An architectural pattern where a repository-hosted, versioned script encapsulates infrastructure and container orchestration calls, exposed via an explicit manual trigger in the continuous integration and continuous deployment control plane and able to be called via authenticated command line or application programming interface requests.

* Good, because it satisfies **Trigger-to-Provisioner Decoupling** by forcing the provisioning script to consume standardized environment variables or flags rather than parsing raw webhook payloads.
* Good, because it satisfies **Baseline Execution Primacy**; subsequent automation options under investigation become pure caller workflows that pass extracted commit hashes into this exact script.
* Good, because it satisfies **Out-of-Band Fault Tolerance** by remaining accessible via direct application programming interface and command line invocation even if source control management webhook dispatchers are degraded.
* Good, because it satisfies **Compute Quota and Cost Containment** by ensuring zero environments are allocated unless an authenticated actor explicitly requests one.
* Neutral, because state management relies on control plane concurrency groups to serialize conflicting manual runs against the same target environment.
* Bad, because it requires maintaining an explicit input-validation pre-flight step to resolve symbolic Git references to commit hashes and verify artifact existence before provisioning begins.

### Option 2: Event-Driven Source Control Management Webhook Provisioning Only

An architectural pattern where ephemeral environments are created, updated, and destroyed exclusively as a side effect of source control management lifecycle events with no decoupled manual invocation contract.

* Good, because it eliminates human invocation latency during standard pull request workflows by binding environment lifecycle directly to branch state transitions.
* Bad, because it violates **Trigger-to-Provisioner Decoupling** by tightly binding the provisioning pipeline to the availability and payload structure of source control management pull request events, preventing deployment of arbitrary commits or un-pulled request branches.
* Bad, because it violates **Out-of-Band Fault Tolerance**; if a webhook payload is dropped during an outage or a teardown event fails, there is no independent mechanism to re-trigger provisioning or force-reap orphaned resources.
* Bad, because it violates **Compute Quota and Cost Containment** by triggering full infrastructure convergence on every Git push to an open pull request, multiplying cloud compute allocation for trivial or work-in-progress commits.

### Option 3: Direct Local Workstation Execution Against Cloud Provider Application Programming Interfaces

An architectural pattern where engineers execute provisioning scripts directly from their local terminal using local credentials to spin up remote ephemeral environments.

* Good, because it satisfies **Out-of-Band Fault Tolerance** at the extreme by removing the continuous integration control plane from the execution path entirely.
* Neutral, because it shares the same underlying script primitive as Option 1, meaning the core logic is reusable if migrated into continuous integration later.
* Bad, because it violates **Authentication and Auditability** by requiring individual developer workstations to hold broad access privileges capable of provisioning compute, networking, and identity sub-resources, bypassing centralized continuous integration audit logs.
* Bad, because it violates **Baseline Execution Primacy**; local workstation execution cannot serve as a callable runtime target for future automated webhook or schedule-driven triggers without re-architecting the execution environment into a remote runner anyway.
* Bad, because local network drops or signal interrupts during execution leave remote infrastructure state locks dangling and cloud resources partially provisioned without an automated runner supervisor to handle cleanup.
