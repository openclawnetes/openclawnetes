# OpenClawNetes

OpenClawNetes is all about running [OpenClaw](https://openclaw.ai/) on Kubernetes.

## Why?

OpenClaw is an open-source AI assistant that can actually do tasks for you, not just chat — things like sending emails, managing calendars, or automating workflows.

For individuals, this is a productivity boost and typically deployed and used locally. But for teams, it has interesting use-cases too. Mundane chores like periodic reminders, daily/weekly work summaries, and other repetitive tasks are great candidates for off-loading to an AI assistant. This works best when the assistant is run and managed centrally, as part of existing infrastructure.

That's what OpenClawNetes does — it provides everything you need to deploy and run OpenClaw on Kubernetes.

## What's in this repo?

This repo contains Kubernetes manifests, architecture documentation, and operational knowledge for running OpenClaw in production. It is organized by cloud provider:

- **`aws/`** — Architecture, deployment manifests, and failure logs for running on EKS

Each directory is self-contained: you can look at a single provider's directory and get the full picture without cross-referencing anything else.

### Deployment manifests

The `deploy/` directory under each provider contains Kubernetes YAML that you apply directly with `kubectl apply -f`. These cover:

- **Deployments** — OpenClaw application pods with resource requests/limits
- **Services** — Internal and external access to OpenClaw
- **ConfigMaps / Secrets** — Configuration for API keys, model settings, and plugin configuration
- **Ingress** — External access with TLS termination
- **PersistentVolumeClaims** — Storage for conversation history and assistant state
- **RBAC** — ServiceAccounts, Roles, and RoleBindings for least-privilege access

### Architecture docs

Each provider's `ARCHITECTURE.md` describes how the components fit together — what runs where, how networking is set up, which cloud-specific services are used (e.g., ALB, EBS, IAM roles for service accounts on AWS), and the reasoning behind key decisions.

### Failure logs

Each provider's `FAILURES.md` tracks what has gone wrong: incidents, misconfigurations, and lessons learned. This is meant to be a living document that helps others avoid the same mistakes.

## Future plans

- Additional cloud providers (GCP, Azure, on-prem)
- A Kubernetes operator (`operator/`) for declarative management of OpenClaw instances via CRDs
- Monitoring and observability recipes (Prometheus, Grafana)
