# Model Highway Implementation Plan

> **For Hermes:** Treat this document as the master roadmap. Before implementing
> a milestone, create its task-level execution plan under `docs/plans/` with
> bite-sized TDD steps, exact commands, expected failures, and commit boundaries.
>
> **Goal:** Make a heterogeneous fleet of personal computers behave like one
> on-demand local LLM serving system, with a useful CLI delivered before any GUI.
>
> **Architecture:** A Rust control plane exposes a versioned management API and a
> deliberately limited OpenAI-compatible gateway. Rust node agents report runtime
> and model observations and receive allowlisted lifecycle commands over an
> authenticated outbound connection. The CLI is the first client of the
> management API. A later Tauri desktop client provides the full cross-platform
> GUI, macOS menu-bar surface, and Linux tray surface without duplicating
> scheduling or lifecycle policy.
>
> **Initial stack:** Rust, Tokio, Axum, Reqwest, Serde, Clap, tracing,
> HTTP/JSON, Server-Sent Events, SQLite, OpenAI-compatible inference APIs,
> Tailscale, launchd, and systemd. The later desktop stack is Tauri 2 with a
> Svelte/TypeScript frontend.

## How to use this plan

This is a decision-gated roadmap, not permission to implement all milestones in
one pass.

For each milestone:

1. Confirm all prerequisite ADRs and earlier acceptance criteria are complete.
2. Write `docs/plans/NN-<milestone>.md` with exact files and TDD-sized tasks.
3. Implement one vertical behavior at a time.
4. Run the milestone's unit, integration, concurrency, and platform checks.
5. Update `README.md`, operations documentation, and API schemas to describe
   only behavior that actually works.
6. Commit in small logical units. Do not push until explicitly authorized.

No milestone may depend on an unresolved decision listed in an earlier gate.

## Product requirements

### CLI-first requirement

The first useful release surface is `model-highwayctl`. It must exercise the
same versioned management API that later native or web clients use. It must not
read SQLite directly or import control-plane internals.

The first vertical slice must support:

- creating local configuration;
- checking control-plane health;
- adding or enrolling one node;
- listing and describing nodes;
- listing known models and their placements;
- explicitly preparing one node/model pair for service;
- routing one streaming chat-completions request through a ready endpoint;
- showing actionable unavailability and failure reasons.

### Later desktop UI requirement

A full cross-platform GUI is deferred until the CLI and management API are
stable, but it is not an optional product direction. The desktop client starts
as a normal macOS/Linux application and later adds a macOS menu-bar surface and
Linux tray surface. The API must support a client that can:

- list all registered servers, retaining offline entries;
- grey out unreachable or policy-disabled servers;
- show a last-known global model catalog;
- select a server and grey out models unavailable on that server;
- display installed, loading, ready, busy, stale, and failed states;
- show wake and model-start progress;
- explain which node served a request and why.

The desktop client uses Tauri 2 with a Svelte/TypeScript frontend and lives in a
separate repository once the management API is stable. The full window and
quick menu-bar/tray surfaces share one frontend state model and API client.
Small Swift/AppKit or Linux-specific integrations are permitted only where
Tauri does not provide acceptable platform behavior.

### Initial functional scope

The first usable core release should:

- route each request to one node;
- discover node, runtime, and model state through agents;
- retain last-known inventory without treating stale data as ready;
- support explicit node/model preparation through the CLI;
- expose currently routable models through `GET /v1/models`;
- proxy streaming and non-streaming `POST /v1/chat/completions`;
- support LAN and Tailscale paths under one node identity;
- wake eligible machines through direct same-subnet dispatch or an
  authenticated LAN relay;
- start and probe one supported local model runtime;
- authenticate management, agent, relay, and inference actions;
- persist command, lifecycle, configuration, and audit state.

### Explicit non-goals for the first core release

- splitting one model or request across computers;
- tensor or pipeline parallelism across the fleet;
- public internet exposure;
- automatic firmware or BIOS configuration;
- automatic model downloads;
- cloud bursting;
- billing, quotas, or multi-tenant scheduling;
- predictive capacity planning;
- transparent retry after streaming output begins;
- implementing desktop GUI source in the core repository.

## Architecture decisions

These are selected defaults. Milestone 0 records them as ADRs and may change one
only when a feasibility result proves it unsuitable.

### Repository strategy

Use one core repository for:

- control plane;
- node agent;
- CLI;
- protocol and API schemas;
- runtime adapter interfaces;
- conformance and integration tests;
- launchd and systemd packaging.

Use a separate repository for the Tauri desktop application. It consumes a
versioned management API and contains presentation, tray/menu-bar behavior, and
OS integration only. Scheduling, model normalization, authentication decisions,
and lifecycle policy remain in the control plane. Share only a versioned Rust
API-client crate or generated OpenAPI client; do not link the desktop process to
the scheduler or SQLite implementation.

The desktop repository obtains the `api-client` crate without publishing it to
crates.io: it depends on the crate by a Git tag that matches an OpenAPI schema
version (a pinned `git`/`tag` Cargo dependency), and the TypeScript types are
generated from the same tagged `api/openapi.yaml`. ADR-0001 fixes this exact
mechanism — tag scheme, version-to-schema mapping, and whether a generated
client is used instead of the crate — before the desktop repository begins.

### Core implementation language

Use Rust for the control plane, node agent, and CLI in one Cargo workspace so
domain types, protocol envelopes, API clients, tests, and release tooling can be
shared. The async stack is Tokio with Axum for servers and Reqwest for clients;
Serde handles wire formats, Clap the CLI, and tracing structured observability.
Milestone 0 records the minimum supported Rust version (MSRV), SQLite crate and
linkage choice, and confirms builds on macOS and Linux before feature work
begins.

### Management and agent protocol

Use versioned HTTP/JSON under `/api/v1`.

The agent initiates all control connectivity:

1. It enrolls with a short-lived, single-use bootstrap token.
2. It opens an authenticated SSE stream for allowlisted commands.
3. It POSTs heartbeats and atomic inventory snapshots.
4. It POSTs command acknowledgements, results, and errors.
5. It reconnects with bounded exponential backoff and jitter.

Every command envelope includes:

- protocol version;
- command ID and idempotency key;
- node ID;
- command type and typed arguments;
- creation time and deadline;
- configuration revision when relevant.

Every acknowledgement/result includes the command ID, node ID, status, and a
structured error when unsuccessful. Replayed commands with the same idempotency
key return the recorded outcome rather than executing twice. Expired commands
must not execute.

Agent-control SSE is separate from the OpenAI-compatible SSE stream used for
chat completions.

### Client-facing APIs

Expose two distinct API families:

- `/api/v1/...` — management, catalog, lifecycle, diagnostics, and UI/CLI data;
- `/v1/...` — the supported OpenAI-compatible inference subset.

`GET /v1/models` returns models currently routable through the gateway. It does
not expose stale or unavailable placements as if they were ready.

The management API retains last-known state. Initial resources include:

- `GET /api/v1/health`
- `GET /api/v1/nodes`
- `GET /api/v1/nodes/{id}`
- `GET /api/v1/servers`
- `GET /api/v1/catalog/models`
- `POST /api/v1/catalog/refresh`
- `POST /api/v1/bootstrap-tokens`
- `POST /api/v1/lifecycle/prepare`
- `GET /api/v1/lifecycle/operations/{id}`
- `GET /api/v1/events`

The API schema is checked into the core repository as OpenAPI or an equivalent
machine-readable contract. CLI and future UI clients must pass compatibility
and conformance tests against it.

For callers that need to pin inference to a ready node, the gateway accepts the
documented optional `X-Model-Highway-Node` request header. Without it, placement
is automatic. A pinned request is rejected with a structured error if that node
is unauthorized, stale, incompatible, or not ready; it never silently falls
back to another node. The header is an explicit Model Highway extension and does
not alter standard OpenAI response bodies.

### Lifecycle behavior

The first lifecycle workflow is explicit and asynchronous:

```text
prepare(node, model)
  -> operation ID
  -> planned
  -> waking (when needed)
  -> agent-online
  -> runtime-starting
  -> model-loading
  -> ready | failed | cancelled | expired
```

Initially, inference requests route only to ready placements. They do not
silently wait through a cold boot. Automatic request-triggered wake or loading
may be added later as an opt-in policy after client timeout and retry behavior
is proven.

#### Wake decision

Waking is a deliberate lifecycle action, never a side effect of routing. A node
is a valid wake candidate only when **all** of these hold:

1. it is in `sleeping-confirmed`, or it is in `offline-unknown` **and** its node
   policy explicitly enables speculative wake;
2. the node is wake-eligible (WoL configured, not on the never-auto-wake list);
3. a verified direct WoL path or an authorized relay endpoint exists;
4. an authorized operator action or an explicitly enabled policy requested the
   prepare that needs it.

`offline-unknown` on its own is never sufficient: a node that is merely
unreachable is not woken unless its policy opts in to speculative wake, because
an unnecessary wake is a real cost on a personal machine. This predicate is
fixed in ADR-0006 before any wake code is written, so the scheduler, the domain
state machine, and Milestone 6 share one definition.

### Retry and streaming rules

- A client cancellation cancels the selected upstream request.
- No transparent retry or failover occurs after response headers or streamed
  content have been committed to the client.
- Before commitment, a request may try another already-ready placement only
  when the first attempt is known not to have been accepted upstream.
- Lifecycle commands use idempotency keys and durable recorded outcomes.
- Timeout budgets distinguish connection, readiness, model startup, request,
  and total-operation deadlines.

### Network legs

Model Highway treats these as separate paths:

1. **Client to gateway** — LAN or Tailscale.
2. **Agent to control plane** — outbound authenticated HTTP/SSE over LAN or
   Tailscale.
3. **Gateway to model runtime** — a service-specific LAN or Tailscale endpoint.
4. **Control plane to local target** — direct subnet-local Wake-on-LAN when the
   control plane has a verified interface on the target subnet.
5. **Control plane to relay** — authenticated management request when direct
   dispatch is unavailable.
6. **Relay to target** — subnet-local Wake-on-LAN broadcast.

A node can have different endpoints for its agent and each runtime. Store
scheme, host, port, transport, TLS identity, observation time, and service type.
A path is ready only after the relevant authenticated health or readiness probe
succeeds.

Prefer LAN for a specific service when reachability from the control plane or
gateway is verified. Fall back to that service's Tailscale endpoint only before
a request is committed. A remote client reaching the gateway through Tailscale
does not force the gateway to use Tailscale for a node on its LAN.

### Security and trust

- Bootstrap tokens are short-lived and single-use.
- Long-lived agent credentials are bound to a stable node identity.
- Credentials support rotation and revocation.
- Operator, agent, relay, and inference-client authorization are separate.
- Control and inference traffic use TLS or authenticated Tailscale transport;
  LAN assumptions are documented in the threat-model ADR.
- Direct wake requires an authorized operator action, an allowlisted target,
  and an audit event. Relay wake calls additionally require relay
  authentication and per-target authorization.
- Node agents independently enforce local wake, runtime, schedule, and resource
  policy.
- Runtime adapters execute typed allowlisted operations, never arbitrary shell
  text supplied by a remote client.
- Secrets use Keychain or permission-restricted files and are never stored in
  inventory records or logs.
- Enrollment, authorization failures, wake, prepare, start, stop, drain, and
  configuration changes are written to an append-only audit log.
- Prompts and generated content are not logged by default.

## Domain model

Keep identity, placement, observation, and serving state separate.

### Node

Stable identity and policy for one computer:

- node ID and display name;
- OS and architecture;
- agent credential identity;
- configured power and usage policy;
- WoL eligibility, subnet/interface metadata, and relay target when required;
- last successful wake metadata.

### RuntimeServer

One runtime installation or managed server on a node:

- server ID and node ID;
- runtime kind and version;
- service-specific endpoints;
- supported operations;
- current process/readiness observation.

### Model

Global canonical model identity:

- canonical model ID and aliases;
- family and capabilities when known;
- context limit when known.

### ModelPlacement

A model available through one runtime server:

- model ID, server ID, and runtime-local reference;
- quantization and storage metadata when known;
- installed, loaded, and ready observations;
- memory estimate and adapter-reported requirements;
- last successful and last attempted verification time;
- current availability reason.

### InventoryObservation

A timestamped atomic agent report. The control plane retains last-known
observations and derives freshness; it never rewrites historical facts to imply
that an offline node is known to be sleeping.

Important states include:

- `never-checked`
- `online`
- `offline-unknown`
- `stale`
- `disabled`
- `unhealthy`
- `sleeping-confirmed` only when an external observation justifies it
- `waking`
- `runtime-starting`
- `model-loading`
- `ready`
- `busy`
- `draining`
- `failed`

A sleeping node runs no agent, so `sleeping-confirmed` can never be derived from
the agent stream. It is set only from one of these external observations, and the
source is recorded on the observation:

- the node's own agent reported an imminent, policy-driven sleep transition
  before the connection closed (a pre-sleep signal);
- an out-of-band reachability probe (for example a managed-switch port state,
  ARP/neighbor liveness, or a configured presence check) declared for that node
  in configuration confirms it is asleep rather than merely unreachable;
- a prior successful wake for the same node established that it was previously
  asleep, until the next contradicting observation.

Absent any such source, a node that stops answering is `offline-unknown`, never
`sleeping-confirmed`. If a node has no configured sleep-confirmation source, it
is wake-eligible from `offline-unknown` only when its policy explicitly opts in
to speculative wake (see the wake decision below); otherwise it is never woken
automatically.

### Command and LifecycleOperation

Durable, idempotent control records with status, deadline, attempt count,
lease/owner, structured result, and audit references. A control-plane restart
must recover or safely expire in-flight work without issuing duplicate actions.

## Scheduling model

Scheduling has two stages.

### Stage 1: eligibility and lifecycle planning

Apply hard constraints before ranking:

1. exact canonical model or alias resolution;
2. runtime and model compatibility;
3. node policy and requested explicit node constraint;
4. installed placement or approved availability source;
5. sufficient adapter-confirmed capacity when loading is required;
6. wake eligibility per the wake decision when the node is not online, which
   requires `sleeping-confirmed`, or `offline-unknown` with a policy that opts
   in to speculative wake;
7. a verified direct WoL path or known relay endpoint when waking is required.

An offline or sleeping node may be a lifecycle candidate but is never directly
routable. A node in `offline-unknown` without a speculative-wake policy is not a
wake candidate at all.

### Stage 2: ready dispatch

Only fresh, healthy, model-ready placements with an acquired capacity lease can
receive inference traffic. Rank eligible ready placements by:

1. explicit caller node selection;
2. already-loaded state;
3. verified LAN path;
4. queue depth or gateway-owned active request count;
5. configured runtime or quantization preference;
6. stable deterministic tie-breaker.

Every selection and rejection produces a structured reason for CLI/UI display
and logs.

## Model polling and catalog consistency

Agents produce atomic inventory snapshots:

- once at startup;
- after a runtime configuration change;
- periodically with jitter;
- in response to a coalesced refresh command.

Each adapter has a bounded deadline. Failures retain the last successful
snapshot but attach a failed-attempt timestamp and reason. The control plane
uses its own receipt time to derive freshness.

A refresh is single-flight per node/runtime. Repeated refresh requests join the
same operation rather than launching parallel polls. A new successful snapshot
atomically replaces the current placement view; disappeared models remain in
last-known history with an explicit unavailable state.

Exact polling intervals and freshness thresholds are configuration values with
safe defaults defined and tested during the catalog milestone.

## Persistence

SQLite is the initial durable store. Use versioned migrations and isolated
temporary databases in tests.

Persist at least:

- nodes and credential metadata;
- runtime servers and service endpoints;
- models, aliases, and placements;
- inventory observations and freshness;
- configuration and revisions;
- commands, acknowledgements, and results;
- lifecycle operations and job leases;
- capacity leases or reservations;
- append-only audit events.

Transactions and unique constraints enforce idempotency. Acceptance tests cover
concurrent heartbeats, duplicate command delivery, restart during wake/start,
job lease recovery, audit retention, and repeated inventory snapshots.

## Proposed core repository layout

```text
Cargo.toml               # Cargo workspace manifest
Cargo.lock               # Reproducible dependency resolution
apps/
├── control-plane/       # Control plane and inference gateway binary
├── agent/               # Node agent binary
└── cli/                 # model-highwayctl binary
api/
└── openapi.yaml         # Versioned management API contract
crates/
├── agent/               # Agent client, command stream, and reporting
├── api/                 # Management and OpenAI-compatible handlers
├── api-client/          # Versioned management API client
├── auth/                # Enrollment, credentials, and authorization
├── catalog/             # Models, placements, observations, and freshness
├── config/              # Versioned configuration
├── control/             # Scheduling and lifecycle orchestration
├── domain/              # Shared IDs, states, and domain types
├── integration-tests/   # Hermetic multi-process test package
├── observability/       # tracing setup, metrics, and redaction
├── persistence/         # SQLite migrations and repositories
├── protocol/            # Agent command/report envelopes
├── runtime/             # Runtime adapter traits and implementations
├── transport/           # Per-service LAN/Tailscale path selection
└── wol/                 # Direct dispatch, magic packets, and relay client/server
deploy/
├── launchd/
└── systemd/
spikes/
└── agent-transport/      # Milestone-0 feasibility evidence, not core API
docs/
├── PLAN.md
├── OPERATIONS.md
├── adr/
└── plans/
```

Avoid publishing Rust crates to crates.io solely for hypothetical reuse. The
versioned API schema and a narrowly scoped API-client crate consumed by Git
tag—not direct database or scheduler crate access—form the compatibility
boundary for the desktop repository. Spikes must be clearly isolated and must
not become production dependencies.

## Testing strategy

### Default offline suite

The normal suite must not require a real model, GPU, Tailscale network, or WoL
hardware. It includes:

- unit tests;
- protocol and OpenAPI conformance tests;
- SQLite migration and restart tests;
- Loom model tests for concurrency-critical command, lease, and job state;
- optional Miri and sanitizer jobs for suitable crates and supported targets;
- a hermetic multi-process integration harness;
- fake agent, runtime, relay, clock, and network dialer;
- streaming, cancellation, reconnect, replay, and timeout tests;
- CLI subprocess tests against a real test control plane.

### Platform checks

Build the core binaries on supported macOS and Linux targets from the foundation
milestone onward. Service-manager tests validate generated launchd and systemd
configuration without requiring installation in the default suite.

### Opt-in fleet suite

Real-fleet tests cover:

- LAN and Tailscale reachability;
- actual runtime inventory and readiness;
- WoL support per device and sleep state;
- cold boot, runtime startup, and model-load timing;
- node disappearance during a stream;
- credential rotation and revoked-node behavior.

## Milestones

### Milestone 0: Architecture and fleet feasibility gate

**Objective:** Resolve all decisions that would otherwise invalidate the
foundation.

**Files:**

- Create: `docs/adr/0001-repository-and-client-boundaries.md`
- Create: `docs/adr/0002-management-and-agent-protocol.md`
- Create: `docs/adr/0003-network-paths.md`
- Create: `docs/adr/0004-domain-and-model-identity.md`
- Create: `docs/adr/0005-threat-model-and-enrollment.md`
- Create: `docs/adr/0006-lifecycle-and-retry-semantics.md`
- Create: `docs/OPERATIONS.md`
- Create: `docs/plans/00-foundation.md`
- Create: `spikes/agent-transport/` (disposable protocol feasibility code)

**Work:**

- Record Rust and the minimum supported Rust version (MSRV).
- Confirm the selected Rust toolchain is available on representative macOS and
  Linux hosts.
- Select the SQLite crate, migration approach, and static/bundled linkage policy.
- Confirm the outbound HTTP/SSE agent design with a reconnect prototype.
- Inventory runtimes on representative fleet nodes.
- Select the first real runtime adapter.
- Verify client-to-gateway, agent-to-control, and gateway-to-runtime paths.
- Test WoL through the intended direct or relayed path for each eligible
  device.
- Record machines that must never be woken automatically.
- Define the wake predicate and every `sleeping-confirmed` source in ADR-0006,
  including whether any node opts in to speculative wake from `offline-unknown`.
- Record cold boot, runtime startup, and model readiness timings.
- Decide model canonicalization and alias rules.

**Acceptance criteria:**

- All six ADRs are accepted and contain no implementation-blocking questions.
- One agent reconnect prototype demonstrates command delivery and deduplication.
- At least one real runtime can list models and expose a testable endpoint.
- Each target device has a documented WoL result and policy.
- The reconnect prototype runs on a representative host, and macOS/Linux
  toolchain availability is documented before the foundation is initialized.
- Hardware-specific results are documented rather than hidden in scheduler code.

### Milestone 1: Functional CLI vertical slice

**Objective:** Deliver a useful CLI and one end-to-end request before building
broad fleet behavior.

**Files:**

- Create: `Cargo.toml`
- Create: `Cargo.lock`
- Create: `docs/plans/01-cli-vertical-slice.md`
- Create: `api/openapi.yaml`
- Create: `apps/control-plane/Cargo.toml`
- Create: `apps/control-plane/src/main.rs`
- Create: `apps/cli/Cargo.toml`
- Create: `apps/cli/src/main.rs`
- Create: `crates/domain/Cargo.toml`
- Create: `crates/domain/src/lib.rs`
- Create: `crates/api/Cargo.toml`
- Create: `crates/api/src/lib.rs`
- Create: `crates/api/src/health.rs`
- Create: `crates/api/src/nodes.rs`
- Create: `crates/api/src/catalog.rs`
- Create: `crates/api/src/gateway.rs`
- Create: `crates/api/src/lifecycle.rs`
- Create: `crates/auth/Cargo.toml`
- Create: `crates/auth/src/lib.rs`
- Create: `crates/auth/src/operator.rs`
- Create: `crates/control/Cargo.toml`
- Create: `crates/control/src/lib.rs`
- Create: `crates/control/src/prepare.rs`
- Create: `crates/observability/Cargo.toml`
- Create: `crates/observability/src/lib.rs`
- Create: `crates/runtime/Cargo.toml`
- Create: `crates/runtime/src/lib.rs`
- Create: `crates/runtime/src/fake.rs`
- Create: `crates/api-client/Cargo.toml`
- Create: `crates/api-client/src/lib.rs`
- Create: `crates/integration-tests/Cargo.toml`
- Create: `crates/integration-tests/src/lib.rs`
- Create: `crates/integration-tests/tests/cli_vertical_slice.rs`
- Modify: `README.md`

**Task groups for the milestone plan:**

1. Initialize the Cargo workspace, shared lint policy, and platform build matrix.
2. Define the versioned health, node, catalog, and error schemas.
3. Add operator authentication and redacted structured logging.
4. Implement a deterministic fake runtime.
5. Implement an in-memory single-node management service.
6. Implement `config init`, `health`, `nodes add/list/describe`, and
   `models list` through the API client.
7. Implement a minimal `serve <node> <model>` prepare operation against the
   fake runtime and expose its operation state through the API.
8. Proxy one streaming and one non-streaming chat request, including an
   explicitly pinned ready-node request.
9. Add a subprocess integration test proving CLI → API → fake runtime.

**Acceptance criteria:**

- The CLI never accesses internal persistence directly.
- A user can configure one endpoint, see its model, prepare it, and route a
  streaming request.
- Invalid credentials and unavailable models produce stable structured errors.
- Client cancellation reaches the fake upstream.
- Unit, integration, Clippy, rustfmt, and macOS/Linux build checks pass. Loom
  concurrency tests apply only once concurrent shared-memory state exists
  (command, lease, and job state in Milestones 2, 4, and 6); M1 has no such
  state and does not gate on Loom.

### Milestone 2: Durable registry, enrollment, and agent channel

**Objective:** Replace manual/in-memory node state with an authenticated agent
and restart-safe persistence.

**Files:**

- Create: `apps/agent/Cargo.toml`
- Create: `apps/agent/src/main.rs`
- Create: `docs/plans/02-agent-and-registry.md`
- Create: `crates/agent/Cargo.toml`
- Create: `crates/agent/src/lib.rs`
- Create: `crates/agent/src/client.rs`
- Create: `crates/agent/src/command_stream.rs`
- Create: `crates/agent/src/reporter.rs`
- Create: `crates/auth/src/bootstrap.rs`
- Create: `crates/auth/src/nodes.rs`
- Create: `crates/config/Cargo.toml`
- Create: `crates/config/src/lib.rs`
- Create: `crates/persistence/Cargo.toml`
- Create: `crates/persistence/src/lib.rs`
- Create: `crates/persistence/migrations/`
- Create: `crates/persistence/src/nodes.rs`
- Create: `crates/persistence/src/config.rs`
- Create: `crates/persistence/src/commands.rs`
- Create: `crates/persistence/src/audit.rs`
- Create: `crates/protocol/Cargo.toml`
- Create: `crates/protocol/src/lib.rs`
- Create: `crates/protocol/src/commands.rs`
- Create: `crates/integration-tests/tests/agent_reconnect.rs`

**Acceptance criteria:**

- A single-use token enrolls one node and fails on replay.
- Revoked and wrong-node credentials are rejected.
- Heartbeats and inventory snapshots are idempotent.
- Commands reconnect, replay, expire, and deduplicate according to the ADR.
- A control-plane restart preserves node, command, and audit state.
- Configuration revisions survive restart and reject stale conditional writes.
- Temporary outages back off with jitter and never create a tight loop.

### Milestone 3: Catalog semantics and first real runtime adapter

**Objective:** Build the global model catalog and per-server placement model
needed by both the CLI and future UIs.

**Files:**

- Create: `crates/catalog/Cargo.toml`
- Create: `crates/catalog/src/lib.rs`
- Create: `crates/catalog/src/models.rs`
- Create: `docs/plans/03-catalog-and-runtime.md`
- Create: `crates/catalog/src/placements.rs`
- Create: `crates/catalog/src/freshness.rs`
- Create: `crates/catalog/src/refresh.rs`
- Create: `crates/runtime/src/adapters/<selected_adapter>.rs`
- Create: `crates/runtime/tests/<selected_adapter>.rs`
- Create: `crates/persistence/src/catalog.rs`
- Create: `crates/integration-tests/tests/catalog_refresh.rs`
- Modify: `apps/cli/src/main.rs`

**Acceptance criteria:**

- The catalog distinguishes global model identity from node placement.
- It distinguishes installed, loaded, ready, stale, offline, absent, and failed.
- Refresh is bounded, jittered, and single-flight.
- A successful inventory snapshot replaces current observations atomically.
- A failed poll retains last-known state with a visible failure reason.
- `/v1/models` returns only routable models while management endpoints retain
  offline and unavailable entries.
- CLI output covers the exact availability matrix required by the later UI.
- `models refresh` uses the management API and joins an existing in-flight
  refresh for the same node/runtime.

### Milestone 4: Multi-node scheduling and explicit placement control

**Objective:** Route among several ready placements without losing manual user
control.

**Files:**

- Create: `crates/control/src/eligibility.rs`
- Create: `docs/plans/04-multi-node-scheduling.md`
- Create: `crates/control/src/scheduler.rs`
- Create: `crates/control/src/selection_reason.rs`
- Create: `crates/control/src/capacity.rs`
- Create: `crates/persistence/src/leases.rs`
- Create: `crates/integration-tests/tests/multi_node_routing.rs`

**Acceptance criteria:**

- Hard compatibility, policy, and capacity constraints run before ranking.
- Explicit node selection is honored or rejected with an exact reason.
- Automatic selection is deterministic under matched state.
- Stale, unhealthy, and non-ready placements never receive inference traffic.
- Capacity leases prevent over-admission under concurrent requests.
- No failover occurs after streamed output begins.

### Milestone 5: Service-specific LAN and Tailscale paths

**Objective:** Select and diagnose the correct path separately for agent,
runtime, gateway, and relay traffic.

**Files:**

- Create: `crates/transport/Cargo.toml`
- Create: `crates/transport/src/lib.rs`
- Create: `crates/transport/src/endpoints.rs`
- Create: `docs/plans/05-network-paths.md`
- Create: `crates/transport/src/reachability.rs`
- Create: `crates/transport/src/selector.rs`
- Create: `crates/transport/tests/selector.rs`
- Create: `crates/integration-tests/tests/path_fallback.rs`

**Acceptance criteria:**

- Endpoint records include service, transport, TLS identity, and freshness.
- Verified LAN is preferred for the relevant service.
- Tailscale fallback obeys per-attempt and total timeout budgets.
- A reachable agent endpoint never implies runtime readiness.
- Fallback occurs only before request commitment.
- Diagnostics report every attempted path and rejection reason.

### Milestone 6: Durable lifecycle operations and Wake-on-LAN

**Objective:** Prepare an installed model on an eligible node through a visible,
restart-safe operation.

**Files:**

- Create: `crates/wol/Cargo.toml`
- Create: `crates/wol/src/lib.rs`
- Create: `crates/control/src/lifecycle.rs`
- Create: `docs/plans/06-lifecycle-and-wol.md`
- Create: `crates/control/src/jobs.rs`
- Create: `crates/persistence/src/jobs.rs`
- Create: `crates/wol/src/magic_packet.rs`
- Create: `crates/wol/src/direct.rs`
- Create: `crates/wol/src/relay.rs`
- Create: `crates/wol/tests/magic_packet.rs`
- Create: `crates/integration-tests/tests/wake_and_prepare.rs`

**Acceptance criteria:**

- Tests verify exact magic-packet bytes without hardware.
- Direct dispatch is allowed only from a verified target-subnet interface and
  records the authorized operator, target, and result in the audit log.
- Relay calls are authenticated and target-authorized.
- Offline-unknown is not assumed to mean sleeping, and is not woken unless the
  node policy opts in to speculative wake.
- A sleeping-confirmed, wakeable node can be planned but not directly routed.
- `sleeping-confirmed` is only ever set from a source named in ADR-0006, never
  inferred from an agent timeout alone.
- Concurrent prepare requests coalesce into one durable operation.
- Restart recovery does not issue duplicate wake or start commands.
- Failed, cancelled, and expired operations expose actionable reasons.
- CLI displays lifecycle progress and final state.

### Milestone 7: Hardening, packaging, and representative fleet validation

**Objective:** Make the core operable on supported macOS and Linux hosts.

**Files:**

- Create: `deploy/launchd/model-highway-agent.plist`
- Create: `docs/plans/07-hardening-and-packaging.md`
- Create: `deploy/systemd/model-highway-agent.service`
- Create: `scripts/build-release.sh`
- Create: `crates/integration-tests/README.md`
- Modify: `docs/OPERATIONS.md`
- Modify: `README.md`

**Acceptance criteria:**

- launchd and systemd run the agent in the intended user/security context.
- Installation, upgrade, rollback, credential rotation, and removal are
  documented.
- A failure on one node does not make unrelated ready nodes unavailable.
- Real-fleet checks cover LAN, Tailscale, ready, unloaded, sleeping, failed WoL,
  startup timeout, revocation, and mid-stream disappearance.
- Default CI remains hardware-independent.
- README claims match the behavior actually released.

### Milestone 8: UI-ready management contract and desktop client handoff

**Objective:** Freeze the management semantics needed by the separate Tauri
desktop repository without moving orchestration logic into that client.

**Files:**

- Modify: `api/openapi.yaml`
- Create: `docs/plans/08-ui-contract.md`
- Create: `docs/UI_CONTRACT.md`
- Create: `crates/integration-tests/tests/ui_contract.rs`
- Create: `docs/adr/0007-client-version-compatibility.md`

**Acceptance criteria:**

- The contract represents offline servers and last-known models.
- It represents per-server availability and exact grey-out reasons.
- Event streaming covers inventory, connectivity, lifecycle, and request state.
- The versioned Rust API-client crate and generated TypeScript fixtures pass
  conformance tests, both produced from the same tagged `api/openapi.yaml` per
  ADR-0001.
- Compatibility and release rules between core and UI repositories are
  documented.
- The Tauri desktop repository can begin without duplicating scheduler,
  lifecycle, authentication, or catalog logic.

### Milestone 9: Full cross-platform desktop GUI

**Objective:** Deliver the normal-window macOS/Linux application before adding
menu-bar or tray-specific interaction.

**Repository:** `model-highway-desktop` (separate from the core repository)

**Files:**

- Create: `model-highway-desktop/package.json`
- Create: `model-highway-desktop/src/` (Svelte/TypeScript application)
- Create: `model-highway-desktop/src/lib/stores/cluster.ts`
- Create: `model-highway-desktop/src/lib/components/ServerPicker.svelte`
- Create: `model-highway-desktop/src/lib/components/ModelPicker.svelte`
- Create: `model-highway-desktop/src-tauri/Cargo.toml`
- Create: `model-highway-desktop/src-tauri/src/lib.rs`
- Create: `model-highway-desktop/src-tauri/src/api_client.rs`
- Create: `model-highway-desktop/tests/` (frontend and API-contract fixtures)

**Architecture:**

- The Rust Tauri shell owns credential storage, management-API calls, and the
  long-lived management event stream.
- The Svelte frontend receives typed state and events through narrow Tauri
  commands/events.
- The desktop process is a client only. Closing or restarting it must not stop
  the control plane, agents, lifecycle jobs, or inference streams.
- The desktop repository consumes a versioned release of the Rust API-client
  crate or a generated client; it never imports scheduler or persistence crates.

**Acceptance criteria:**

- Signed/development builds launch as normal windows on supported macOS and
  Linux targets.
- The server picker retains offline and disabled nodes with exact grey-out
  reasons.
- The model picker shows the global last-known catalog and per-server
  availability.
- Lifecycle operations display progress, cancellation, failure, and final state.
- Settings and diagnostics cover connectivity, refresh timestamps, selected
  paths, and structured rejection reasons.
- Frontend tests and desktop API-contract fixtures run without a real fleet.
- Quitting the desktop client leaves the independently managed core services
  running.

### Milestone 10: macOS menu bar and Linux tray

**Objective:** Add quick-control surfaces to the same Tauri desktop application
without creating separate product logic or background services.

**Files:**

- Create: `model-highway-desktop/src-tauri/src/tray.rs`
- Create: `model-highway-desktop/src-tauri/src/platform/macos.rs`
- Create: `model-highway-desktop/src-tauri/src/platform/linux.rs`
- Create: `model-highway-desktop/src/lib/components/QuickControl.svelte`
- Create: `model-highway-desktop/tests/tray/`

**Acceptance criteria:**

- macOS uses an adaptive template menu-bar icon and can open quick controls or
  the full dashboard.
- Linux provides a tray/status icon and menu on the explicitly supported
  desktop environments.
- The documented Linux support matrix records desktop-environment limitations,
  left/right-click behavior, and any required status-notifier packages.
- Quick controls reuse the same server/model stores and lifecycle API as the
  full window.
- Tray/menu-bar actions cannot bypass authorization or local node policy.
- Hiding or closing the last window preserves the tray client when configured;
  quitting the tray exits only the desktop client, not Model Highway services.
- Platform-specific Swift/AppKit or Linux code is isolated behind narrow Rust
  interfaces and is added only when Tauri behavior is insufficient.

## Remaining product decisions

These do not block the foundation and can be resolved at the named milestone:

1. Exact default polling and freshness intervals — Milestone 3.
2. Default idle-unload policy — Milestone 6 or later.
3. Whether automatic request-triggered prepare is ever enabled — after
   Milestone 6 evidence.
4. Exact Tauri Linux tray behavior and desktop-environment support matrix —
   during desktop-client implementation.
5. Whether additional OpenAI-compatible endpoints such as Responses or
   embeddings enter a later release.

## Verification commands

These become authoritative when the Cargo workspace exists:

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
cargo build --workspace --all-targets
```

Milestone plans must add exact targeted commands and expected output. Real GPU,
model, Tailscale, service installation, and WoL checks remain explicit opt-in
operations documented in `docs/OPERATIONS.md`.
