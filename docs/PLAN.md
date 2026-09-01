# Model Highway Implementation Plan

> **For Hermes:** Treat this document as the master roadmap. Before implementing
> a milestone, create its task-level execution plan under `docs/plans/` with
> bite-sized TDD steps, exact commands, expected failures, and commit boundaries.
>
> **Goal:** Make a heterogeneous fleet of personal computers behave like one
> on-demand local LLM serving system, with a useful CLI delivered before any GUI.
>
> **Architecture:** A Go control plane exposes a versioned management API and a
> deliberately limited OpenAI-compatible gateway. Go node agents report runtime
> and model observations and receive allowlisted lifecycle commands over an
> authenticated outbound connection. The CLI is the first client of the
> management API. Native user interfaces may be added later without duplicating
> scheduling or lifecycle policy.
>
> **Initial stack:** Go, HTTP/JSON, Server-Sent Events, SQLite,
> OpenAI-compatible inference APIs, Tailscale, launchd, and systemd.

## How to use this plan

This is a decision-gated roadmap, not permission to implement all milestones in
one pass.

For each milestone:

1. Confirm all prerequisite ADRs and earlier acceptance criteria are complete.
2. Write `docs/plans/NN-<milestone>.md` with exact files and TDD-sized tasks.
3. Implement one vertical behavior at a time.
4. Run the milestone's unit, integration, race, and platform checks.
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

### Later UI requirement

A GUI is deferred until the CLI and management API are stable, but it is not an
optional product direction. The API must support a later client that can:

- list all registered servers, retaining offline entries;
- grey out unreachable or policy-disabled servers;
- show a last-known global model catalog;
- select a server and grey out models unavailable on that server;
- display installed, loading, ready, busy, stale, and failed states;
- show wake and model-start progress;
- explain which node served a request and why.

The intended first native client is a macOS menu-bar application. Its source
should live in a separate repository once the management API is stable. Linux
UI work may use a separate native or webview client later.

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
- implementing macOS or Linux GUI source in the core repository.

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

Use separate repositories for native UIs as they materialize. UIs consume a
versioned management API and contain presentation and OS integration only.
Scheduling, model normalization, authentication decisions, and lifecycle policy
remain in the control plane.

### Core implementation language

Use Go for the control plane, node agent, and CLI so protocol types, tests, and
release tooling can be shared. Milestone 0 records the minimum supported Go
version and confirms builds on macOS and Linux before feature work begins.

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
6. wake eligibility when the node is not online;
7. a verified direct WoL path or known relay endpoint when waking is required.

An offline or sleeping node may be a lifecycle candidate but is never directly
routable.

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
cmd/
├── model-highway/       # Control plane and inference gateway
├── model-highway-agent/ # Node agent
└── model-highwayctl/    # Management CLI
api/
└── openapi.yaml         # Versioned management API contract
internal/
├── agent/               # Agent client, command stream, and reporting
├── api/                 # Management and OpenAI-compatible handlers
├── auth/                # Enrollment, credentials, and authorization
├── catalog/             # Models, placements, observations, and freshness
├── config/              # Versioned configuration
├── control/             # Scheduling and lifecycle orchestration
├── observability/       # Structured logs, metrics, tracing hooks
├── persistence/         # SQLite migrations and repositories
├── protocol/            # Agent command/report envelopes
├── runtime/             # Runtime adapter interface and implementations
├── transport/           # Per-service LAN/Tailscale path selection
└── wol/                 # Direct dispatch, magic packets, and relay client/server
pkg/
└── apiclient/           # Generated or thin public management client if needed
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
tests/
└── integration/         # Hermetic multi-process harness
```

Avoid exporting Go packages solely for hypothetical reuse. The versioned API
schema, not direct database or internal package access, is the compatibility
boundary for separate repositories. Spikes must be clearly isolated and must
not become production dependencies.

## Testing strategy

### Default offline suite

The normal suite must not require a real model, GPU, Tailscale network, or WoL
hardware. It includes:

- unit tests;
- protocol and OpenAPI conformance tests;
- SQLite migration and restart tests;
- `go test -race ./...` where supported;
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

- Record Go and its minimum supported version.
- Confirm the selected Go toolchain is available on representative macOS and
  Linux hosts.
- Confirm the outbound HTTP/SSE agent design with a reconnect prototype.
- Inventory runtimes on representative fleet nodes.
- Select the first real runtime adapter.
- Verify client-to-gateway, agent-to-control, and gateway-to-runtime paths.
- Test WoL through the intended direct or relayed path for each eligible
  device.
- Record machines that must never be woken automatically.
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

- Create: `go.mod`
- Create: `docs/plans/01-cli-vertical-slice.md`
- Create: `api/openapi.yaml`
- Create: `cmd/model-highway/main.go`
- Create: `cmd/model-highwayctl/main.go`
- Create: `internal/api/health.go`
- Create: `internal/api/nodes.go`
- Create: `internal/api/catalog.go`
- Create: `internal/api/gateway.go`
- Create: `internal/api/lifecycle.go`
- Create: `internal/auth/operator.go`
- Create: `internal/control/prepare.go`
- Create: `internal/runtime/runtime.go`
- Create: `internal/runtime/fake.go`
- Create: `pkg/apiclient/client.go`
- Create: `tests/integration/cli_vertical_slice_test.go`
- Modify: `README.md`

**Task groups for the milestone plan:**

1. Initialize the module and platform build matrix.
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
- Unit, race, integration, and macOS/Linux build checks pass.

### Milestone 2: Durable registry, enrollment, and agent channel

**Objective:** Replace manual/in-memory node state with an authenticated agent
and restart-safe persistence.

**Files:**

- Create: `cmd/model-highway-agent/main.go`
- Create: `docs/plans/02-agent-and-registry.md`
- Create: `internal/agent/client.go`
- Create: `internal/agent/command_stream.go`
- Create: `internal/agent/reporter.go`
- Create: `internal/auth/bootstrap.go`
- Create: `internal/auth/nodes.go`
- Create: `internal/persistence/migrations/`
- Create: `internal/persistence/nodes.go`
- Create: `internal/persistence/config.go`
- Create: `internal/persistence/commands.go`
- Create: `internal/persistence/audit.go`
- Create: `internal/protocol/commands.go`
- Create: `tests/integration/agent_reconnect_test.go`

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

- Create: `internal/catalog/models.go`
- Create: `docs/plans/03-catalog-and-runtime.md`
- Create: `internal/catalog/placements.go`
- Create: `internal/catalog/freshness.go`
- Create: `internal/catalog/refresh.go`
- Create: `internal/runtime/<selected_adapter>.go`
- Create: `internal/runtime/<selected_adapter>_test.go`
- Create: `internal/persistence/catalog.go`
- Create: `tests/integration/catalog_refresh_test.go`
- Modify: `cmd/model-highwayctl/main.go`

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

- Create: `internal/control/eligibility.go`
- Create: `docs/plans/04-multi-node-scheduling.md`
- Create: `internal/control/scheduler.go`
- Create: `internal/control/selection_reason.go`
- Create: `internal/control/capacity.go`
- Create: `internal/persistence/leases.go`
- Create: `tests/integration/multi_node_routing_test.go`

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

- Create: `internal/transport/endpoints.go`
- Create: `docs/plans/05-network-paths.md`
- Create: `internal/transport/reachability.go`
- Create: `internal/transport/selector.go`
- Create: `internal/transport/selector_test.go`
- Create: `tests/integration/path_fallback_test.go`

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

- Create: `internal/control/lifecycle.go`
- Create: `docs/plans/06-lifecycle-and-wol.md`
- Create: `internal/control/jobs.go`
- Create: `internal/persistence/jobs.go`
- Create: `internal/wol/magic_packet.go`
- Create: `internal/wol/direct.go`
- Create: `internal/wol/relay.go`
- Create: `internal/wol/magic_packet_test.go`
- Create: `tests/integration/wake_and_prepare_test.go`

**Acceptance criteria:**

- Tests verify exact magic-packet bytes without hardware.
- Direct dispatch is allowed only from a verified target-subnet interface and
  records the authorized operator, target, and result in the audit log.
- Relay calls are authenticated and target-authorized.
- Offline-unknown is not assumed to mean sleeping.
- A sleeping-confirmed, wakeable node can be planned but not directly routed.
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
- Create: `tests/integration/README.md`
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

### Milestone 8: UI-ready management contract and native client handoff

**Objective:** Freeze the management semantics needed for separate native UI
repositories without moving orchestration logic into those clients.

**Files:**

- Modify: `api/openapi.yaml`
- Create: `docs/plans/08-ui-contract.md`
- Create: `docs/UI_CONTRACT.md`
- Create: `tests/integration/ui_contract_test.go`
- Create: `docs/adr/0007-client-version-compatibility.md`

**Acceptance criteria:**

- The contract represents offline servers and last-known models.
- It represents per-server availability and exact grey-out reasons.
- Event streaming covers inventory, connectivity, lifecycle, and request state.
- A generated or thin native client passes conformance fixtures.
- Compatibility and release rules between core and UI repositories are
  documented.
- The macOS menu-bar repository can begin without duplicating scheduler,
  lifecycle, authentication, or catalog logic.

## Remaining product decisions

These do not block the foundation and can be resolved at the named milestone:

1. Exact default polling and freshness intervals — Milestone 3.
2. Default idle-unload policy — Milestone 6 or later.
3. Whether automatic request-triggered prepare is ever enabled — after
   Milestone 6 evidence.
4. The first Linux GUI toolkit — after the UI contract is stable.
5. Whether additional OpenAI-compatible endpoints such as Responses or
   embeddings enter a later release.

## Verification commands

These become authoritative when the Go module exists:

```bash
go test ./...
go test -race ./...
go vet ./...
go build ./...
```

Milestone plans must add exact targeted commands and expected output. Real GPU,
model, Tailscale, service installation, and WoL checks remain explicit opt-in
operations documented in `docs/OPERATIONS.md`.
