# ADR-0003: roeld is a daemon (not a cron job or one-shot)

## Status

Accepted

## Context

`roel` has two consumption patterns:

1. **One-shot** — run recommendations on demand, output results, exit. Suitable for manual runs, CI/CD pipelines, and testing.
2. **Continuous** — run as a long-running service, ingest metrics on a schedule, produce recommendations periodically, expose results via HTTP API. Suitable for continuous monitoring and dashboard integration.

The question is whether the continuous pattern should be a daemon (`roeld`), a cron job, or a one-shot script.

## Decision

`roeld` is a daemon — a long-running process that:

- Ingests PCP metrics on a configurable schedule (e.g., daily)
- Computes digests and recommendations
- Exposes results via an HTTP API
- Manages its own lifecycle (startup, shutdown, health checks)

The CLI (`roel`) remains a one-shot tool for manual runs and testing.

This follows the Unix convention of suffixing daemon processes with `d` (e.g., `sshd`, `httpd`, `systemd`).

## Alternatives Considered

### Cron job

A cron job that runs `roel` on a schedule. This is simpler but has no HTTP API, no state management, no health checks, and no way to query results on demand. It also requires the cron infrastructure to be configured and maintained.

### One-shot with external scheduler

A one-shot `roel` invoked by an external scheduler (e.g., Kubernetes CronJob, systemd timer). This is more flexible but pushes scheduling, state management, and API exposure to the external system. The `roel` tool becomes a library rather than a product.

### Daemon with embedded scheduler

`roeld` runs as a daemon with an embedded scheduler. This is the chosen approach — it keeps the scheduling logic inside `roeld`, exposes an HTTP API for on-demand queries, and manages its own lifecycle.

## Consequences

- **HTTP API** — `roeld` exposes endpoints for querying recommendations, triggering manual runs, and checking health.
- **State management** — `roeld` manages its own state (last run, last results, errors). This may require a database or in-memory store (see Open Questions in the design document).
- **Lifecycle management** — `roeld` handles graceful shutdown, signal handling, and health checks.
- **Scheduling** — `roeld` has an embedded scheduler for periodic ingestion and recommendation runs.
- **Operational complexity** — a daemon is more operationally complex than a cron job. It requires process supervision (systemd, etc.) and monitoring.

## References

- [roel design document](../design/roel-design.md) — Section 4.4: Output
- [ros-ocp-backend](https://github.com/pgarciaq/ros-ocp-backend) — Follows the same daemon pattern (rosocp processor)
