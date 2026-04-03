# Failures and Lessons Learned

This document tracks failures, incidents, and lessons learned while running OpenClaw on Kubernetes in AWS.

## Template

When adding a new entry, use this format:

### [Date] - Brief title

**What happened:**

**Root cause:**

**Resolution:**

**Lessons learned:**

---

### [2026-04-01] - Tool execution fails due to Docker sandbox in Kubernetes

**What happened:**

OpenClaw was given a task to post a message to Slack using a webhook URL. It generated a `curl` command and attempted to execute it. Instead of running the command directly, the default sandbox mode tried to pull a Docker image and execute the command inside a Docker container. This failed repeatedly because the gateway was running inside a Kubernetes pod with no access to a Docker daemon.

**Root cause:**

OpenClaw's default sandbox mode (`docker`) assumes a Docker socket is available for isolating tool execution. In a Kubernetes environment, pods do not have access to a Docker daemon, so all sandboxed tool executions fail.

**Resolution:**

Set `agents.defaults.sandbox.mode` to `"off"` in `openclaw.json`. This causes the gateway to execute tools directly in the same process/node where it is running.

**Lessons learned:**

- The default sandbox mode is not compatible with Kubernetes deployments out of the box.
- Setting `sandbox.mode: "off"` restores tool execution but introduces an availability risk: a misbehaving tool (e.g. a runaway process, excessive memory use) can destabilize or crash the gateway itself, since there is no isolation boundary.
- A better long-term fix would be to delegate tool execution to a separate sidecar or worker node, keeping the gateway process insulated from tool failures.

---

<!-- Add entries below -->
