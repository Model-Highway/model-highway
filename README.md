# Model Highway

Self-hosted orchestration for serving local LLMs across a heterogeneous fleet of
personal computers.

> **Status:** Architecture and feasibility planning. The repository currently
> contains documentation only; the runtime is not implemented yet.

## Why

A home lab may contain Macs and Linux machines with different CPUs, GPUs,
runtimes, model files, network paths, and power states. Using them as one
inference resource should not require remembering which machine has a model,
whether it is awake, or which endpoint is reachable.

Model Highway will provide a stable CLI and API for discovering the fleet,
preparing a selected node and model, and routing inference through a ready local
runtime. It will prefer private LAN paths where verified, use Tailscale for
remote access or fallback paths, and use an authenticated LAN relay for
Wake-on-LAN when needed.

## Delivery order

The project is deliberately **CLI first**:

1. validate the architecture and actual fleet;
2. deliver a functional CLI vertical slice against one runtime;
3. add authenticated agents, durable state, and model polling;
4. add multi-node scheduling, LAN/Tailscale selection, and lifecycle control;
5. stabilize the management API;
6. build a cross-platform Tauri desktop client in a separate repository;
7. add macOS menu-bar and Linux tray surfaces to that desktop client.

The CLI is the first client of the same versioned management API that the future
desktop client will use. It will not bypass the API to read the database or
reuse control-plane internals.

See [`docs/PLAN.md`](docs/PLAN.md) for decisions, milestone gates, acceptance
criteria, and the testing strategy.

## Planned capabilities

- Register computers as nodes in a personal inference fleet.
- Poll node health, runtime servers, and installed model placements.
- Retain last-known offline and stale state without treating it as ready.
- Expose a versioned management API for the CLI and later UIs.
- Expose a supported OpenAI-compatible inference subset.
- Explicitly prepare a selected node/model pair through a durable operation.
- Route requests only to fresh, healthy, model-ready placements.
- Support automatic deterministic placement while preserving explicit node
  selection.
- Prefer verified LAN endpoints for each service.
- Fall back to service-specific Tailscale endpoints before request commitment.
- Send Wake-on-LAN directly on the target subnet or through an authenticated
  relay on that LAN.
- Start, stop, drain, and unload model-serving processes through allowlisted
  runtime adapters.
- Show which node handled a request and why it was selected or rejected.

The first release will route each request to one machine. It will not split one
model or request across computers.

## CLI-first user experience

The CLI grows through the milestones. Its initial and planned commands include:

```text
model-highwayctl config init
model-highwayctl health
model-highwayctl nodes add ...
model-highwayctl nodes list
model-highwayctl nodes describe <id>
model-highwayctl models list
model-highwayctl models refresh
model-highwayctl serve <node> <model>
```

`serve` starts an explicit asynchronous lifecycle operation and returns an
operation ID. The CLI follows transitions such as:

```text
planned
  -> waking
  -> agent-online
  -> runtime-starting
  -> model-loading
  -> ready
```

Initially, inference requests route only to ready placements. They will not
silently remain open through a cold boot or model load. Automatic
request-triggered wake may be considered later as an opt-in policy after timeout
and retry behavior is proven.

## Architecture

```text
                         Client to gateway
                     LAN or Tailscale, authenticated
                                   |
                                   v
                     +---------------------------+
                     | Control plane             |
                     | management API + catalog  |
                     | scheduler + lifecycle     |
                     | OpenAI-compatible gateway |
                     +-------------+-------------+
                                   |
                +------------------+------------------+
                |                                     |
      Agent outbound HTTP/SSE              Gateway to runtime
        LAN or Tailscale                   LAN or Tailscale
                |                                     |
        +-------v-------+                     +-------v-------+
        | Node agent    |---- local control ->| Model runtime |
        +---------------+                     +---------------+

Control plane -- direct subnet-local WoL ------------------+
                                                          +--> sleeping node
Control plane -- authenticated relay -- subnet-local WoL --+
```

Networking is modeled per service. An address that reaches an agent does not
imply that a model runtime is reachable. A remote client may reach the gateway
through Tailscale while the gateway still uses a verified LAN path to the
selected runtime.

### Control plane

Maintains durable node, runtime, model, command, lifecycle, configuration, and
audit state. It exposes two API families:

- `/api/v1/...` for management, catalog, lifecycle, events, and diagnostics;
- `/v1/...` for the supported OpenAI-compatible inference subset.

`GET /v1/models` contains currently routable models. Last-known offline models
and per-server availability belong to the management catalog instead. Inference
placement is automatic unless an authorized caller uses the documented optional
`X-Model-Highway-Node` header to pin a ready node; a failed pin never silently
falls back to another node.

### Node agent

Runs on each participating computer. It:

- enrolls with a short-lived, single-use bootstrap token;
- opens an authenticated outbound SSE command stream;
- posts heartbeats and atomic inventory snapshots;
- executes typed, allowlisted runtime operations;
- reports command acknowledgements and results;
- enforces local power, schedule, and resource policy.

Commands use IDs, deadlines, idempotency keys, durable outcomes, and defined
reconnect/replay semantics. The control plane does not become a general remote
shell.

### Runtime adapters

Provide a common interface over local serving backends. Candidates include
llama.cpp, Ollama, MLX LM, and vLLM. A fleet feasibility milestone selects one
real adapter before implementation expands to others.

The catalog separates:

- global model identity;
- a model placement on a specific runtime server;
- timestamped inventory observations;
- currently loaded and ready serving state.

This lets the CLI and future UIs distinguish installed, unloaded, ready, stale,
offline, absent, and failed models.

### Wake-on-LAN

Tailscale does not directly deliver a subnet-local Wake-on-LAN broadcast to a
sleeping machine. Model Highway sends the packet directly when the control
plane has a verified interface on the target subnet. Otherwise it uses an
authenticated always-on relay on that LAN.

A sleeping or offline node is never directly routable. It may be selected as a
lifecycle candidate only when policy, capacity, runtime support, a direct or
relayed wake path, and Wake-on-LAN eligibility are satisfied.
`offline-unknown` is not treated as proof that a node is sleeping.

## Future desktop UI

After the CLI and management API are stable, the next product surface is a full
cross-platform desktop application for macOS and Linux. It should be able to:

- show all registered servers and grey out offline or disabled ones;
- show the last-known global model catalog;
- select a server and grey out models unavailable there;
- display refresh times and failure reasons;
- follow wake and model-start progress;
- provide full settings and diagnostics views.

The desktop application uses Tauri 2 with a Svelte/TypeScript frontend. After
the full dashboard is stable, the same application adds a macOS menu-bar surface
and Linux tray surface for quick status, server/model selection, prepare, and
dashboard access. It lives in a separate repository and consumes the versioned
management API. Scheduling, authentication, catalog normalization, and lifecycle
behavior remain in the core service to avoid duplication. Small Swift/AppKit or
Linux-specific integrations are allowed only when Tauri cannot provide adequate
platform behavior.

## Initial stack and repository boundary

- **Rust** for the control plane, node agent, and CLI.
- **Tokio**, **Axum**, **Reqwest**, **Serde**, **Clap**, and **tracing** for the
  asynchronous core, APIs, CLI, wire formats, and observability.
- **HTTP/JSON** for the versioned management protocol.
- **Server-Sent Events** for outbound agent commands and management events.
- **SQLite** for registry, observations, configuration, commands, jobs, leases,
  and append-only audit events.
- **OpenAI-compatible HTTP/SSE** for the supported inference surface.
- **Tailscale** for private remote connectivity and service-specific fallback.
- **launchd** and **systemd** for macOS and Linux service management.
- **Tauri 2 + Svelte/TypeScript** for the later full GUI and menu-bar/tray
  surfaces.

This core repository contains the daemon, agent, CLI, API schema, adapters,
packaging, and conformance tests. The desktop repository shares the API contract
and a narrowly scoped versioned API-client crate rather than Rust scheduler
crates or the SQLite schema.

## Security principles

- Management, agent, relay, and inference actions are authenticated.
- Bootstrap credentials are short-lived, single-use, and node-bound.
- Long-lived credentials support rotation and revocation.
- Model and agent endpoints are private by default.
- Tailscale ACLs constrain remote access.
- LAN and Tailscale trust assumptions are documented explicitly.
- Lifecycle commands are typed and allowlisted.
- Secrets are kept out of inventories, logs, and committed configuration.
- Wake, prepare, start, stop, drain, enrollment, and authorization failures are
  auditable.
- Prompt and generated content are not logged by default.
- Each node retains a local policy boundary even when the control plane requests
  an operation.

## Testing approach

The default suite will be hardware-independent and include unit, concurrency,
protocol-conformance, SQLite restart, CLI subprocess, and hermetic multi-process
integration tests. Fake agents, runtimes, relays, clocks, and network dialers
will exercise reconnects, replay, polling freshness, streaming cancellation,
path fallback, job coalescing, and restart recovery.

Real GPU, runtime, Tailscale, service installation, and Wake-on-LAN checks are
explicit opt-in fleet tests documented separately.

## Planned core repository layout

```text
model-highway/
├── Cargo.toml              # Rust workspace manifest
├── Cargo.lock              # Reproducible dependency resolution
├── apps/                   # Control plane, node agent, and CLI binaries
├── api/                    # Versioned management API schema
├── crates/                 # Domain, protocol, persistence, control, runtime
│   └── integration-tests/  # Hermetic multi-process harness
├── deploy/                 # launchd and systemd service definitions
├── docs/                   # Roadmap, ADRs, operations, milestone plans
├── scripts/                # Build and release tooling
└── spikes/                 # Isolated feasibility evidence
```

## Development status

The repository is still documentation-only. The next implementation step is
Milestone 0 in [`docs/PLAN.md`](docs/PLAN.md): architecture ADRs and feasibility
checks on representative macOS and Linux nodes before the Rust foundation is
created.
