# Architecture: OpenClaw on AWS/Kubernetes

## Overview

OpenClaw runs inside a dedicated `openclaw` namespace on an EKS cluster. The architecture separates the **gateway** (API server, session management) from the **node** (tool execution). The gateway serves an HTTP API on port 18789, backed by persistent storage for conversation history and assistant state. Nodes connect to the gateway via WebSocket and execute tools on its behalf. An auto-approve sidecar on the gateway pod automatically approves node pairing requests, enabling hands-free node registration.

## Diagram

![Architecture Diagram](assets/architecture.png)

The source for this diagram is in [assets/architecture.mmd](assets/architecture.mmd) and can be regenerated with:

```bash
npx @mermaid-js/mermaid-cli -i aws/assets/architecture.mmd -o aws/assets/architecture.png -b white --scale 2
```

## Components

### Gateway pod (Deployment)

The gateway deployment runs a pod with an init container and two main containers:

1. **Init container** (`busybox:1.37`) — copies `openclaw.json` and `AGENTS.md` from the ConfigMap into the persistent home directory at `/home/node/.openclaw/`. This runs once before the main containers start.

2. **Gateway container** (`ghcr.io/openclaw/openclaw:latest`) — the main OpenClaw process, running `openclaw gateway run`. It serves an HTTP API on port 18789 with token-based authentication. Tool execution is delegated to connected nodes via the `tools.exec.host: "node"` configuration.

3. **Auto-approve sidecar** (`ghcr.io/openclaw/openclaw:latest`) — a lightweight sidecar that:
   - On startup, cleans up stale disconnected nodes (disconnected for more than 10 minutes)
   - Continuously polls for pending device pairing requests and auto-approves them (every 5 seconds)

   This enables nodes to connect and start executing tools without manual approval.

The deployment uses `strategy: Recreate` because the pod mounts a ReadWriteOnce PVC, which cannot be attached to multiple nodes simultaneously.

### Node pod (StatefulSet)

The node runs as a StatefulSet with a reconnect loop:

1. **Init container** (`busybox:1.37`) — copies `exec-approvals.json` from the node approvals ConfigMap, granting the node full tool execution permissions without interactive approval prompts.

2. **Node container** (`ghcr.io/openclaw/openclaw:latest`) — runs `openclaw node run` in a loop, connecting to the gateway at `openclaw.openclaw.svc.cluster.local:18789` via WebSocket. If disconnected, it retries every 10 seconds. Each node identifies itself using its pod hostname (`--display-name "$(hostname)"`).

The node uses `emptyDir` for its home directory (not a PVC) — nodes are ephemeral workers with no persistent state. The StatefulSet provides stable pod identity for display names.

### ConfigMap: `openclaw-config`

Contains two files:

- **`openclaw.json`** — gateway configuration: bind address, auth mode (token), agent definitions, model provider settings (Anthropic), node support (`nodes.allowCommands`, `tools.exec.host`), and feature flags (Slack, cron disabled).
- **`AGENTS.md`** — agent instructions file copied into the workspace.

### ConfigMap: `openclaw-node-approvals`

Contains `exec-approvals.json` — pre-approves all tool execution on the node with `security: "full"` and `ask: "off"`. This allows the node to execute tools autonomously without interactive consent prompts.

### Secrets

Two secrets are referenced as environment variables:

| Secret | Key | Used by | Purpose |
|---|---|---|---|
| `openclaw-gateway-token` | `OPENCLAW_GATEWAY_TOKEN` | Gateway, auto-approve sidecar, node | Bearer token for authenticating to the gateway API |
| `claude-api-key` | `ANTHROPIC_API_KEY` | Gateway, node | API key for Anthropic's Claude models |

These can be created manually, or via ExternalSecrets, Vault, SOPS, etc. See `secrets.yaml.example`.

### Service

A `ClusterIP` service exposes the gateway on port 18789 within the cluster. The node connects to this service internally. There is no Ingress configured — external access is cluster-internal only. To expose externally, add an Ingress or LoadBalancer service.

### Storage

A 10Gi `PersistentVolumeClaim` (`openclaw-data`) is mounted at `/home/node/.openclaw` on the gateway pod. This path is OpenClaw's home directory — it's where the application stores its configuration, conversation history, assistant state, and workspace files. Without persistent storage, all of this would be lost every time the pod restarts.

The mount path is `/home/node/.openclaw` specifically because the OpenClaw container image runs as the `node` user (UID 1000), and the application expects its data directory at `~/.openclaw`. The init container also writes configuration files (copied from the ConfigMap) into this directory before the gateway starts.

On EKS, the PVC defaults to a gp2/gp3 EBS volume.

## Security

All pods follow a hardened security posture:

- Runs as non-root (UID/GID 1000)
- Read-only root filesystem
- No privilege escalation
- All Linux capabilities dropped
- Seccomp profile: RuntimeDefault
- Service account token not mounted (`automountServiceAccountToken: false`)
- `/tmp` is an emptyDir (writable but ephemeral)

## Networking

- **Inbound**: ClusterIP service on port 18789 (cluster-internal only)
- **Internal**: Node connects to gateway via WebSocket on the ClusterIP service
- **Outbound**: HTTPS to `api.anthropic.com/v1` for Claude API calls; OTLP/HTTP to the configured metrics endpoint
- No Ingress is configured by default — the gateway is not exposed to the internet

## Observability

OpenClaw's built-in `diagnostics-otel` plugin can export OpenTelemetry metrics via OTLP (http/protobuf) to any OTLP-compatible endpoint. **Disabled by default** — enable it when you have a collector endpoint ready.

### How to enable

1. In `config.yaml`, set `diagnostics.enabled` and `diagnostics.otel.enabled` to `true` and update the endpoint.
2. In `deployment.yaml`, set `OTEL_METRICS_EXPORTER` to `"otlp"` and `OTEL_EXPORTER_OTLP_ENDPOINT` to your collector.

Point these at any OTLP-compatible receiver — a Datadog agent, Grafana Alloy, a standalone OTel Collector, etc. An example OTel Collector config is provided in `otel-collector-config.yaml`.

### Available metrics

| Metric | Type | Description |
|---|---|---|
| `openclaw.tokens` | Counter | Token usage by type (input, output, cache_read, cache_write) |
| `openclaw.cost.usd` | Counter | Estimated model costs in USD |
| `openclaw.message.queued` | Counter | Messages queued for processing |
| `openclaw.message.processed` | Counter | Messages processed by outcome |
| `openclaw.run.attempt` | Counter | Agent run attempt counts |
| `openclaw.session.state` | Counter | Session state transitions |
| `openclaw.session.stuck` | Counter | Sessions stuck in processing |
| `openclaw.run.duration_ms` | Histogram | Agent run duration |
| `openclaw.context.tokens` | Histogram | Context window size and usage |
| `openclaw.message.duration_ms` | Histogram | Message processing duration |
| `openclaw.queue.depth` | Histogram | Queue depth |
| `openclaw.queue.wait_ms` | Histogram | Queue wait time before execution |

### Configuration

OTel is configured in `openclaw.json` under `diagnostics.otel`. Only metrics are enabled by default — traces and logs can be turned on if needed.

## Current Limitations

- **Single node replica.** The node StatefulSet currently runs 1 replica. Scaling to multiple replicas is planned for improved availability.
- **Local access requires port-forwarding.** The gateway service is ClusterIP-only, so accessing OpenClaw from a local machine is not straightforward. You need to port-forward the service to localhost first:

  ```bash
  kubectl port-forward -n openclaw svc/openclaw 18789:18789
  ```

  Then access it at `http://localhost:18789`. There is no Ingress or LoadBalancer configured by default. Setting up proper external access (Ingress with TLS, ALB, etc.) is a future improvement.

## AWS Services

| Service | Usage |
|---|---|
| **EKS** | Kubernetes control plane and worker nodes |
| **EBS** | Persistent storage for the gateway PVC (default gp2/gp3 storage class) |
| **ECR** (optional) | Can mirror `ghcr.io/openclaw/openclaw:latest` for faster pulls |
| **IAM** (optional) | IRSA (IAM Roles for Service Accounts) if using ExternalSecrets with AWS Secrets Manager |
