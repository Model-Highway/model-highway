# Model Highway Implementation Plan

> **Goal:** Make a heterogeneous fleet of personal computers behave like one on-demand local LLM serving system.
>
> **Architecture:** A control plane exposes a stable OpenAI-compatible gateway, maintains a model-aware node registry, and schedules requests. A lightweight node agent runs on each computer, manages local serving runtimes, and reports health and inventory. LAN and Tailscale provide alternate network paths; Wake-on-LAN is used only to transition eligible nodes from sleep to agent-ready.
>
> **Tech stack:** Go, HTTP/JSON, Server-Sent Events, SQLite, OpenAI-compatible inference APIs, Tailscale, launchd, and systemd.

## Scope and principles

### Initial scope

The first usable release should:

- route a request to one node at a time;
- discover node health and model inventory from agents;
- support LAN and Tailscale addresses under one node identity;
- wake eligible machines through a configured LAN relay;
- start and probe a local model server;
- expose `/v1/models` and streaming `/v1/chat/completions`;
- provide a CLI for registration, status, and troubleshooting;
- keep model servers private and authenticate all control actions.

### Explicit non-goals for the first release

- splitting one model across multiple computers;
- tensor or pipeline parallelism across the fleet;
- public internet exposure;
- automatic firmware/BIOS configuration;
- cloud bursting;
- billing, quotas, or multi-tenant scheduling;
- automatic model downloads;
- sophisticated predictive capacity planning.

### Design principles

1. **One node per request first.** Replication and failover are simpler and more useful than distributed inference for the initial system.
2. **Runtime-neutral core.** Scheduling should reason about capabilities and readiness, not runtime-specific process details.
3. **Explicit state.** A stale inventory must never look equivalent to a healthy, ready endpoint.
4. **Private by default.** LAN and Tailscale are the normal paths; public exposure is not required.
5. **Explainable scheduling.** The system should record why a node was selected, woken, or rejected.
6. **Safe lifecycle control.** The control plane may manage approved model-server commands, but it must not become a general remote shell.
7. **Recoverable operations.** A lost agent, failed wake, or runtime crash should become a visible state with a bounded timeout.

## Proposed repository layout

```text
cmd/
├── model-highway/       # Control-plane server and gateway
├── model-highway-agent/ # Node agent
└── model-highwayctl/    # Administrative CLI
internal/
├── api/                 # HTTP handlers and OpenAI-compatible gateway
├── control/             # Scheduler and lifecycle orchestration
├── inventory/           # Nodes, models, runtimes, and freshness
├── persistence/         # SQLite schema and repositories
├── protocol/            # Agent/control messages and versioning
├── runtime/             # Runtime adapter interface and implementations
├── transport/           # LAN/Tailscale address selection
└── wol/                 # Magic-packet dispatch and relay support
pkg/
└── protocol/            # Shared types only if external clients need them
docs/
├── PLAN.md
└── adr/                 # Architecture decision records as decisions solidify
tests/
└── integration/         # Multi-process and network-bound tests
```

The exact package layout may change after the first Go module is initialized; the boundaries above are the intended responsibilities, not an excuse to build abstractions before they are needed.

## Milestones

### Milestone 0: Establish the project foundation

**Objective:** Turn the documentation scaffold into a buildable Go project with repeatable local development checks.

**Files:**

- Create: `go.mod`
- Create: `cmd/model-highway/main.go`
- Create: `cmd/model-highway-agent/main.go`
- Create: `cmd/model-highwayctl/main.go`
- Create: `internal/README.md` or package documentation where useful
- Modify: `README.md`

**Work:**

- Choose and record the minimum supported Go version.
- Add a minimal executable for each planned binary.
- Add `go test ./...` as the first verification command.
- Keep the binaries intentionally non-functional beyond a clear startup/help message.

**Acceptance criteria:**

- `go test ./...` passes.
- Each binary builds with `go build ./...`.
- README development instructions describe the commands that actually work.

### Milestone 1: Define the domain and wire protocol

**Objective:** Establish stable concepts for nodes, models, runtimes, health, and agent commands before implementing scheduling.

**Files:**

- Create: `internal/protocol/types.go`
- Create: `internal/protocol/messages.go`
- Create: `internal/protocol/types_test.go`
- Create: `docs/adr/0001-agent-protocol.md`

**Work:**

Define versioned messages for:

- node registration;
- heartbeat;
- model inventory report;
- runtime inventory report;
- readiness and health probes;
- start/stop/unload commands;
- command acknowledgements and failures.

Model records should include a canonical ID, aliases, runtime, quantization when known, context limit when known, memory estimate when known, and last verification time. Node records should include stable ID, OS/architecture, capabilities, LAN addresses, Tailscale address, WoL metadata, and local policy.

**Acceptance criteria:**

- Invalid required fields are rejected in tests.
- Protocol messages carry a version.
- Unknown optional fields can be ignored for forward compatibility.
- No message permits arbitrary shell command text as a control primitive.

### Milestone 2: Implement persistence and node registry

**Objective:** Store node state and model inventory with explicit freshness and lifecycle state.

**Files:**

- Create: `internal/persistence/sqlite.go`
- Create: `internal/persistence/schema.sql`
- Create: `internal/persistence/nodes.go`
- Create: `internal/persistence/nodes_test.go`
- Create: `internal/inventory/state.go`
- Create: `internal/inventory/state_test.go`

**Work:**

- Use SQLite with migrations or an equivalent versioned schema approach.
- Store node identity separately from current observations.
- Track `last_seen`, inventory freshness, readiness, and failure reason.
- Make updates idempotent so repeated heartbeats do not create duplicate nodes or models.
- Represent stale/offline state explicitly rather than deleting records immediately.

**Acceptance criteria:**

- A node can register, update its heartbeat, replace inventory, and be marked stale.
- Restarting the control plane preserves state.
- Tests run against an isolated temporary database.
- Concurrent or repeated updates do not corrupt registry state.

### Milestone 3: Build the node agent and control-plane connection

**Objective:** Have an agent register itself and maintain a heartbeat over an authenticated connection.

**Files:**

- Create: `internal/agent/client.go`
- Create: `internal/agent/heartbeat.go`
- Create: `internal/agent/agent_test.go`
- Create: `internal/api/agent_handlers.go`
- Create: `internal/api/agent_handlers_test.go`

**Work:**

- Implement outbound agent registration and heartbeat.
- Add an explicit node credential/bootstrap token flow.
- Report OS, architecture, hostname, configured addresses, and basic resource information.
- Use bounded request timeouts and exponential backoff.
- Make reconnect behavior safe after a control-plane restart.

**Acceptance criteria:**

- A fresh agent appears in the registry.
- The control plane can distinguish a fresh heartbeat from stale state.
- Invalid credentials are rejected.
- A temporary network outage causes reconnect attempts without a tight loop.

### Milestone 4: Add runtime adapters and model inventory

**Objective:** Discover models and manage a local server through a runtime-neutral interface.

**Files:**

- Create: `internal/runtime/runtime.go`
- Create: `internal/runtime/fake.go`
- Create: `internal/runtime/fake_test.go`
- Create: `internal/runtime/ollama.go` or `internal/runtime/llamacpp.go`
- Create: `internal/runtime/<adapter>_test.go`
- Modify: `internal/agent/heartbeat.go`

**Initial adapter decision:** Start with the runtime that gives the best testable OpenAI-compatible path across the user's current machines. The implementation should evaluate Ollama, llama.cpp, MLX LM, and vLLM during the feasibility spike; do not implement all four before the first end-to-end route works.

**Adapter interface:**

- enumerate installed models;
- return normalized metadata;
- start a configured model server;
- stop a managed server;
- report readiness;
- unload or evict a model when supported.

**Acceptance criteria:**

- Inventory reports distinguish installed from loaded/ready models.
- The fake adapter supports deterministic unit tests without a model download.
- A real adapter can probe a local test endpoint and normalize its model list.
- Runtime failures include an actionable reason and do not crash the agent.

### Milestone 5: Implement the request gateway and scheduler

**Objective:** Route OpenAI-compatible model requests to a ready node.

**Files:**

- Create: `internal/api/gateway.go`
- Create: `internal/api/gateway_test.go`
- Create: `internal/control/scheduler.go`
- Create: `internal/control/scheduler_test.go`
- Create: `internal/control/selection_reason.go`

**Work:**

Implement:

- `GET /v1/models` from the aggregated, fresh inventory;
- `POST /v1/chat/completions` with streaming and non-streaming paths;
- model alias resolution;
- deterministic node selection;
- upstream timeout and cancellation propagation;
- selection reasons in structured logs and response metadata where appropriate.

Initial selection order:

1. exact model/alias match;
2. fresh and healthy node;
3. already-loaded model;
4. same-LAN path;
5. lowest queue depth or active request count;
6. sufficient declared memory/VRAM;
7. configured runtime/quantization preference.

**Acceptance criteria:**

- A request reaches a fake ready node and returns its response.
- Streaming bytes are forwarded without buffering the whole completion.
- Unknown models produce a clear client error.
- Unhealthy or stale nodes are never selected.
- A cancelled client request cancels the upstream request.
- Scheduler tests prove deterministic selection and explainable rejection.

### Milestone 6: Add LAN/Tailscale path selection

**Objective:** Select a reachable address for a node without duplicating node identities.

**Files:**

- Create: `internal/transport/address.go`
- Create: `internal/transport/address_test.go`
- Create: `docs/adr/0002-network-paths.md`

**Work:**

- Store LAN and Tailscale addresses on the same node record.
- Prefer LAN when the control plane can verify local reachability.
- Fall back to Tailscale when LAN is unavailable.
- Avoid requiring public port forwarding.
- Add diagnostics that report which path was attempted and why.

**Acceptance criteria:**

- Address selection is deterministic under mocked network conditions.
- A failed LAN path can fall back to Tailscale within a bounded timeout.
- The system never treats an unverified address as ready solely because it is configured.

### Milestone 7: Implement Wake-on-LAN and startup orchestration

**Objective:** Wake an eligible node and wait for it to become ready before routing.

**Files:**

- Create: `internal/wol/magic_packet.go`
- Create: `internal/wol/magic_packet_test.go`
- Create: `internal/wol/relay.go`
- Create: `internal/control/lifecycle.go`
- Create: `internal/control/lifecycle_test.go`
- Create: `docs/adr/0003-wake-on-lan.md`

**Work:**

- Validate MAC addresses before sending packets.
- Support direct LAN broadcast from the control plane when possible.
- Support a configured always-on LAN relay when the control plane is remote.
- Model `sleeping`, `waking`, `booting`, `agent-online`, `runtime-starting`, `ready`, and failure states.
- Poll agent heartbeat and runtime readiness with a total startup deadline.
- Prevent duplicate wake/start jobs for the same node and model.
- Respect node power policy and an explicit “do not wake” setting.

**Acceptance criteria:**

- Unit tests verify the exact magic-packet bytes without requiring real hardware.
- A simulated sleeping node transitions through wake and readiness states.
- Failed wake or startup produces a bounded, actionable error.
- Concurrent requests coalesce on one startup job rather than starting duplicate servers.

### Milestone 8: Add CLI, observability, and operator workflows

**Objective:** Make the system diagnosable without requiring the web UI.

**Files:**

- Modify: `cmd/model-highwayctl/main.go`
- Create: `internal/observability/logging.go`
- Create: `internal/observability/metrics.go`
- Create: `docs/OPERATIONS.md`

**CLI commands:**

- `nodes list`
- `nodes describe <id>`
- `nodes register`
- `models list`
- `models refresh`
- `wake <node>`
- `serve <node> <model>`
- `drain <node>`
- `health`

**Acceptance criteria:**

- An operator can identify why a model is unavailable.
- Logs include request ID, node ID, model ID, selected network path, lifecycle state, and failure reason.
- Metrics cover request count/latency, queue time, wake time, model startup time, failures, and active nodes.
- Sensitive credentials and prompt contents are not logged by default.

### Milestone 9: Package and validate on the target fleet

**Objective:** Exercise the end-to-end workflow on representative macOS and Linux machines.

**Files:**

- Create: `deploy/launchd/model-highway-agent.plist`
- Create: `deploy/systemd/model-highway-agent.service`
- Create: `scripts/build-release.sh`
- Create: `tests/integration/README.md`
- Modify: `README.md`
- Modify: `docs/OPERATIONS.md`

**Validation matrix:**

- always-on control-plane host;
- macOS node;
- Linux node;
- LAN-only request;
- remote Tailscale request;
- already-ready model;
- installed-but-not-loaded model;
- sleeping node with working WoL;
- failed WoL;
- runtime startup timeout;
- node disappearing during a streamed response.

**Acceptance criteria:**

- Installation and service management are documented for supported hosts.
- End-to-end tests use fake runtimes by default and real runtimes only in explicitly configured environments.
- A failure in one node does not make unrelated healthy nodes unavailable.
- The README no longer describes implemented behavior that the release does not support.

## Feasibility spike checklist

Before committing to the first runtime adapter, validate the following on the actual fleet:

- Which machines can receive Wake-on-LAN from the intended relay?
- Which machines are reachable over Tailscale while on or awake?
- Which runtime is already installed or easiest to deploy on each OS?
- Can the runtime expose an OpenAI-compatible endpoint?
- How long do cold boot, runtime startup, and model loading take?
- What metadata is available for installed models?
- How should the system estimate memory/VRAM requirements?
- Which machines should never be woken automatically?

Record hardware-specific findings in `docs/OPERATIONS.md` or an ADR rather than encoding them as hidden scheduler behavior.

## Security requirements

- Authenticate node registration and all control-plane commands.
- Use TLS or Tailscale transport for control and inference traffic.
- Keep model endpoints bound to private interfaces unless explicitly configured otherwise.
- Use Tailscale ACLs to limit who can reach the control plane and agents.
- Never accept arbitrary shell commands from the client-facing API.
- Store credentials outside model inventory and committed configuration files.
- Audit wake, start, stop, drain, and configuration changes.
- Make wake and usage policy enforceable per node.

## Open decisions

1. Confirm Go as the implementation language after the foundation spike.
2. Choose REST/JSON plus SSE versus gRPC for the agent protocol.
3. Choose the first real runtime adapter: Ollama, llama.cpp, MLX LM, or vLLM.
4. Decide whether the control plane owns WoL directly or always delegates to a LAN relay.
5. Define the bootstrap flow for adding a node without copying long-lived credentials.
6. Decide how model aliases are authored and reconciled across runtimes.
7. Define whether client requests may trigger model loading by default or require an explicit policy.
8. Define safe retry semantics for non-streaming and streaming requests.
9. Set default idle-unload and auto-wake policies.
10. Decide when a web UI is justified beyond the CLI and structured logs.

## Verification commands

These commands become authoritative as implementation lands:

```bash
go test ./...
go vet ./...
go build ./...
```

Network- and hardware-dependent checks should be opt-in and documented separately. Tests that require a real model, a specific GPU, a Tailscale network, or Wake-on-LAN hardware must not be part of the default offline test suite.
