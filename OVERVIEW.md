# AX Repository Overview

## Purpose

**AX** (module `github.com/google/ax`) is a declarative, high-throughput orchestrator for **autonomous AI agent workloads** running on a cluster. You describe an agentic task in YAML (what to run, which repos/tools it needs, what network it may reach, which LLM to use), and AX:

1. **Sandboxes** it: each task runs as an isolated actor on [Agent Substrate](https://github.com/agent-substrate/substrate), with CPU and memory limits.
2. **Wires up its workspace**: clones Git repos, configures MCP servers and skill registries, and can optionally hand a plain-language `goal` to an agent that finishes environment setup.
3. **Fences its network**: applies an egress allowlist of hosts and ports.
4. **Runs it at scale**: state lives in Redis, not etcd, so the control plane can handle millions of short-lived tasks.

It borrows the Kubernetes model: `apiVersion`/`kind` manifests and a `kubectl`-style CLI (`ax apply`, `get`, `describe`, `watch`, `delete`). It also adds agent-specific verbs: `suspend`, `resume` and `ssh`.

> ⚠️ The project is pre-stable (`v1alpha1`). Concepts and APIs may still change in breaking ways.

## Uses

| Use case | How |
|---|---|
| Run untrusted agent code in isolation | `Task` with image, command, env, resource limits |
| Start agents "warm" with repos, MCP tools and skills already in place | `Workspace`, bound to one or more tasks |
| Restrict an agent to e.g. only your LLM provider and Git host | `Gateway` egress allowlist |
| Manage LLM provider, model, parameters and API-key secret in one place | `Model` (default: Google / `gemini-3.8-flash`, key in secret `gemini-api-secret`) |
| Pause an idle agent and continue later | `ax suspend task` / `ax resume task` (checkpoints actor state) |
| Debug a live agent | `ax ssh <task>` (needs `spec.debug: true`) |
| Fan out large trees of sub-tasks | Tasks are small and cheap, so an agent can create many of them |
| Custom agent runtime | Build your own image that embeds the `runner` package |

A task moves through phases (`Running`, `Suspended`, `Failed`, `Terminating`, …). Its conditions are `WorkspaceReady`, `GatewayReady`, and `Ready`; wait on `Ready`.

## Architecture

```
 ax apply -f task.yaml   (CLI; auto-tunnels via kubectl port-forward)
          │ gRPC
          ▼
      ax-server  ── validate, store, publish event ──►  Redis
   (gRPC + /healthz :8080)                  (hashes + Streams + PubSub)
                                                     │ XREADGROUP
                                                     ▼
                                              ax-controller  (N replicas)
                                                     │ gRPC Control API
                                                     ▼
                                              Agent Substrate
                                (atespaces, actors, workers, egress policy)
                                                     │
                                                     ▼
                                   Task container → ax-task-runner
                     (metadata server, guest services, workspace setup, runs command)
```

Every resource lives in an **atespace**, a namespace-like scope. The default atespace is `default`.

## Repository structure

```
.
├── cmd/                      # Main entry points (one binary each)
│   ├── ax/                   # Developer CLI (~1.2k LOC, kubectl-shaped)
│   ├── ax-server/            # Stateless gRPC API server
│   ├── ax-controller/        # Reconciliation workers
│   └── ax-task-runner/       # In-container entrypoint (+ antigravity_bootstrap.py)
├── runner/                   # Public package: the in-sandbox run loop (embeddable)
├── pkg/apis/v1alpha1/        # Public API: ax.proto, generated gRPC/protobuf Go, types
├── internal/
│   ├── server/               # gRPC service implementation (Task/Gateway/Workspace/Model CRUD, Watch, Suspend/Resume)
│   ├── controller/           # Redis-stream worker + TaskReconciler
│   ├── substrate/            # Agent Substrate Control API client
│   ├── store/                # Store interface
│   │   ├── redis/            #   production Redis backend
│   │   └── memory/           #   in-memory backend (tests)
│   ├── workspace/            # Workspace setup (git, MCP, skills) + goal-driven Planner
│   ├── model/                # LLM client (Google provider), K8s secret resolution
│   ├── metadata/             # Metadata HTTP server inside the sandbox
│   ├── guest/                # Guest-services client used by `ax ssh`
│   └── tunnel/               # kube-context detection + background port-forward tunnels (~/.ax/tunnels)
├── deploy/                   # K8s manifests: redis.yaml, ax-server.yaml, ax-controller.yaml (ko)
├── examples/                 # simple.yaml (bare Task), task.yaml (Task+Workspace+Gateway+Model)
├── docs/                     # concepts, manifests, sandbox, runner, networking, development, roadmap
├── Dockerfile.task-runner    # python:3.12-slim + git/curl/ssh + google-antigravity + runner binary
├── .ko.yaml                  # ko build config (chainguard static base images)
├── Makefile                  # build / install / test / deploy targets
├── demo.sh                   # End-to-end lifecycle demo
├── DESIGN.md                 # Architecture + gRPC API reference
└── .github/workflows/        # CI: go mod tidy check, build all binaries, go test
```

## Main entry points

| Entry point | File | What it does |
|---|---|---|
| **`ax`** CLI | `cmd/ax/main.go` | Parses a subcommand (`apply`, `get`, `describe`, `watch`, `suspend`, `resume`, `delete`, `ssh`, `ctx`, `tunnel`, `version`). Resolves the control plane from the active kube context, `--server` or `$AX_SERVER`, and talks gRPC. |
| **`ax-server`** | `cmd/ax-server/main.go` | Serves the `ax.v1alpha1.AX` gRPC service and `/healthz` over HTTP/1 plus h2c on `:8080`, backed by the Redis store. Flags/env: `--addr`/`ADDR`, `--redis-addr`/`REDIS_ADDR`, `REDIS_PASSWORD`. |
| **`ax-controller`** | `cmd/ax-controller/main.go` | Consumes the Redis stream (consumer group `ax-controllers`) and reconciles tasks against Agent Substrate. Flags: `--substrate-endpoint` (default `api.ate-system.svc.cluster.local:443`), TLS/token options, and `--template`/`--template-atespace` for the default ActorTemplate. |
| **`ax-task-runner`** | `cmd/ax-task-runner/main.go` | PID 1 inside each task container. Loads the Task from `AX_TASK_YAML` and the Workspaces from `AX_WORKSPACES_YAML`, or from `--task-file`/`--workspace-file` for local runs. Then it calls `runner.Run`. |
| **`runner.Run`** | `runner/runner.go` | Starts the metadata and guest server (port 80), runs first-boot workspace setup, then supervises the task command. It stays alive afterwards so the sandbox can still be inspected. |
| **gRPC API** | `pkg/apis/v1alpha1/ax.proto` | Get/List/Update/Delete for Tasks, Gateways, Workspaces and Models, plus `SuspendTask`, `ResumeTask` and the streaming `WatchTask`. |

## Key dependencies

| Dependency | Role |
|---|---|
| Go **1.27.1** | Language / toolchain (`go.mod`) |
| `github.com/agent-substrate/substrate`, `.../env` | Sandboxed actor runtime and Control API that AX orchestrates |
| `github.com/redis/go-redis/v9` | State store and work queue (Streams + PubSub) |
| `google.golang.org/grpc`, `google.golang.org/protobuf` | Control-plane API and guest services |
| `gopkg.in/yaml.v3` | Manifest parsing |
| **External tools** | Kubernetes cluster + `kubectl`, [`ko`](https://ko.build/), Docker/Podman, a container registry, a reachable Agent Substrate install |
| **Task image** | Python 3.12 + `google-antigravity` (the agent used for goal-driven workspace bootstrap) |

## How to run

### Build and test locally (no cluster needed)

```bash
make build          # -> bin/ax, bin/ax-controller, bin/ax-server
make test           # go test -v ./...  (uses in-memory store + mock Substrate)
make install        # installs `ax` into $(go env GOPATH)/bin
# or: go install github.com/google/ax/cmd/ax@latest
```

All test packages passed on the current `main` (controller, metadata, model, server, tunnel, workspace, v1alpha1, runner).

### Deploy the control plane to Kubernetes

Prerequisites: a cluster with Agent Substrate reachable from it, `ko`, and a registry your cluster can pull from.

```bash
make deploy AX_IMAGE_REPO=<your-registry>   # redis + ax-controller + ax-server into ns ax-system
# Optional: custom task-runner image
make push-task-runner TASK_RUNNER_REPO=<your-registry>/ax-task-runner
```

### Use it

```bash
ax apply -f examples/simple.yaml        # minimal Task
ax apply -f examples/task.yaml          # Task + Workspace + Gateway + Model
ax get tasks
ax watch task <name>                    # live phase/condition changes
ax ssh <name> -- ls -la /workspace      # requires spec.debug: true
ax suspend task <name> && ax resume task <name>
ax delete task <name>
./demo.sh                               # full lifecycle demo
```

Global flags: `-a/--atespace` (default `default`), `-n/--namespace` (default `ax-system`), `--context`, `--server`.

### Run the task runner locally

```bash
go run ./cmd/ax-task-runner --port 8081 \
  --task-file my-task.yaml --workspace-file my-workspace.yaml
```

## Further reading

- `README.md`: quick start and CLI reference
- `DESIGN.md`: architecture and full RPC reference
- `docs/concepts.md`, `docs/manifests.md`: resource model and annotated YAML
- `docs/sandbox.md`, `docs/runner.md`: what runs inside the container, and how to build a custom runner
- `docs/networking.md`: reaching tasks via the atenet router
- `docs/roadmap.md`: spec stabilization, new actor architecture, dynamic environments, governance
