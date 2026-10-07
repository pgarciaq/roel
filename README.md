# roel: Resource Optimization for Enterprise Linux

`roel` provides resource optimization recommendations for **standalone RHEL machines** — bare-metal processes, Podman containers, quadlets, Docker containers, and cgroups. It uses PCP (Performance Co-Pilot) as the metrics data source and reuses the proven `librobne` recommendation engines from [ros-ocp-backend](https://github.com/pgarciaq/ros-ocp-backend).

## Why roel exists

`ros-ocp-backend` handles OpenShift rightsizing. `roel` handles the same problem for standalone RHEL machines that are not part of an OpenShift cluster. Same recommendation math, different data source.

## Architecture

```
RHEL Node
├── pmcd (PCP daemon)
├── pmlogger (archival)
├── pmdaproc (processes)
├── pmdapodman (Podman containers)
├── pmdadocker (Docker containers)
├── pmdasystemd (quadlets)
└── pmdanvidia (GPU)

roel-metrics-adapter
├── PMAPI client → reads from pmcd
├── Computes daily percentiles
└── Emits DigestRow JSON

libroel
├── librobne engines (node, gpu, container, digest, savings)
├── disk engine (new — IOPS/throughput-based disk type)
└── RHEL workload abstraction (process, container, quadlet, cgroup)

roel (CLI)     — one-shot: load → digest → recommend → output
roeld (daemon) — long-running: HTTP API + scheduled ingestion
```

## Components

| Component | Description |
|-----------|-------------|
| `libroel` | Go module — RHEL-specific engines, PCP adapter, workload abstraction |
| `roel` | CLI — one-shot recommendation runs |
| `roeld` | Daemon — HTTP API, scheduled ingestion, continuous monitoring |
| `librobne` | Shared recommendation engine library (from ros-ocp-backend) |

## Relationship to existing projects

| Project | Relationship |
|---------|-------------|
| [ros-ocp-backend](https://github.com/pgarciaq/ros-ocp-backend) | Sibling service — OpenShift rightsizing |
| [librobne](https://github.com/pgarciaq/ros-ocp-backend/tree/main/librobne) | Shared engine library — reused by roel |
| [robne](https://github.com/pgarciaq/ros-ocp-backend/tree/main/cmd/robne) | Sibling CLI — OpenShift recommendations |
| [PCP](https://pcp.readthedocs.io/) | Data source — RHEL-native monitoring |

## Documentation

- [Conceptual design document](docs/design/roel-design.md)
- [Architecture Decision Records](docs/adr/README.md)

## Status

Early design phase. See the [design document](docs/design/roel-design.md) for architecture, data flow, and open questions.
