# Model Highway

Self-hosted orchestration for serving local LLMs across a fleet of personal computers.

> **Status:** Early design. The repository currently contains project documentation only; the runtime is not implemented yet.

## Why

A powerful home lab often has several computers with different CPUs, GPUs, operating systems, model files, and power states. Using them as one inference resource should not require manually remembering which machine has which model or whether that machine is awake.

Model Highway will provide one stable way to request a model while the system discovers the best available computer, wakes it when possible, starts the required serving runtime, and routes the request over the local network or Tailscale.

## Planned capabilities

- Register computers as nodes in a personal inference fleet.
- Poll node health, hardware, runtimes, and installed models.
- Expose a stable OpenAI-compatible API.
- Select a node based on model compatibility, reachability, load, and policy.
- Prefer direct LAN communication when available.
- Use Tailscale for remote access without exposing model servers publicly.
- Send Wake-on-LAN packets through an always-on device when an eligible node is asleep.
- Wait for the node and model server to become ready before routing traffic.
- Start, stop, and unload model-serving processes through runtime adapters.
- Show which node handled a request and why it was selected.

## How a request will work

1. A client requests a model through the gateway.
2. The scheduler finds a healthy node advertising that model.
3. If the best node is already ready, the request is routed immediately.
4. If the node is asleep, a LAN-side relay sends a Wake-on-LAN packet.
5. The node agent reconnects and starts the required model runtime.
6. The gateway polls readiness and forwards the request once the model is available.
7. Streaming output is returned through the same stable gateway endpoint.

The first version will route each request to one machine. It will not split a single model or request across multiple computers.

## Architecture

```text
                           +----------------------+
  OpenAI-compatible       |  Control plane       |
  client ---------------->|  API + scheduler     |
                           |  registry + gateway  |
                           +----------+-----------+
                                      |
                    LAN or Tailscale | authenticated control
                                      |
             +------------------------+------------------------+
             |                         |                        |
       +-----v-----+             +-----v-----+            +-----v-----+
       | Node agent |             | Node agent |            | WoL relay |
       | Runtime A  |             | Runtime B  |            | (optional)|
       +-----+------+             +-----+------+            +-----------+
             |                          |
       Local model server         Local model server
```

### Control plane

Maintains node and model state, selects a serving node, exposes the client API, manages lifecycle commands, and records health and routing information.

### Node agent

Runs on each participating computer. It reports capabilities and model inventory, maintains a heartbeat, manages local model servers, and exposes readiness and health state.

### Runtime adapters

Provide a common interface over local serving backends. Planned integrations include llama.cpp, Ollama, MLX LM, and vLLM.

### Networking

LAN and Tailscale are treated as two paths to the same node identity. Wake-on-LAN is a separate power-management path and may require an always-on relay on the target subnet.

## Proposed initial stack

- **Go** for the control plane, node agent, and CLI.
- **HTTP/JSON** for the first control protocol and **SSE** for streaming status/events.
- **SQLite** for the initial registry, configuration, job state, and audit log.
- **OpenAI-compatible HTTP** for the client-facing inference gateway.
- **Tailscale** for private remote connectivity.
- **launchd** and **systemd** service definitions for macOS and Linux.

The implementation plan records the assumptions and sequencing: [`docs/PLAN.md`](docs/PLAN.md).

## Security principles

- Model servers are private by default.
- Nodes authenticate to the control plane.
- Tailscale ACLs constrain remote access.
- Lifecycle commands are explicit and allowlisted; the gateway does not execute arbitrary shell commands.
- Wake, start, stop, and routing actions are auditable.
- Each node can enforce local power and usage policies.

## Repository status

This is the initial project scaffold. Documentation is being established before implementation begins.

Planned top-level structure:

```text
model-highway/
├── cmd/          # Executables: control plane, node agent, CLI
├── internal/     # Private application packages
├── pkg/          # Shared public protocol/types, if needed
├── docs/         # Design and implementation documentation
└── README.md
```

## Development

Development instructions will be added with the first executable milestone. Until then, see [`docs/PLAN.md`](docs/PLAN.md) for the implementation sequence and acceptance criteria.
