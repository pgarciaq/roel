# ADR-0002: PCP is the metrics data source

## Status

Accepted

## Context

`roel` needs historical usage metrics from standalone RHEL machines to compute rightsizing recommendations. The data source must be:

- **RHEL-native** — available on RHEL without external dependencies
- **Lightweight** — suitable for production systems
- **Historical** — must retain enough data for percentile-based recommendations (at least 7-30 days)
- **Comprehensive** — must cover CPU, memory, disk IOPS/throughput, and GPU metrics
- **Workload-aware** — must distinguish between processes, containers, quadlets, and cgroups

Three options were considered: Prometheus exporters, Performance Co-Pilot (PCP), and a custom robne-metrics adapter.

## Decision

PCP (Performance Co-Pilot) is the metrics data source for `roel`. Specifically, `pcp-zeroconf` provides zero-configuration setup on RHEL nodes.

PCP meets all requirements:

| Requirement | PCP |
|-------------|-----|
| RHEL-native | Built into RHEL — `yum install pcp-zeroconf` |
| Lightweight | `pmcd` + `pmlogger` is minimal overhead |
| Historical | `pmlogger` archives to disk with configurable retention |
| Comprehensive | CPU, memory, disk IOPS/throughput, GPU, network, per-process, per-container |
| Workload-aware | `pmdaproc`, `pmdapodman`, `pmdadocker`, `pmdasystemd`, `cgroup.*` |

Key PMDAs used:

| PMDA | Metrics | Used for |
|------|---------|----------|
| `pmdalinux` | `kernel.all.cpu.*`, `mem.util.*`, `disk.dev.*` | System-level CPU, memory, disk |
| `pmdaproc` | `proc.psinfo.*` | Per-process metrics |
| `pmdapodman` | Podman container metrics | Container-level metrics |
| `pmdadocker` | Docker container metrics | Container-level metrics |
| `pmdasystemd` | `systemd.unit.*` | Quadlet service units |
| `pmdanvidia` | `nvidia.*` | GPU metrics |
| `pmdaopenmetrics` | Scrapes `/metrics` | Custom/DCGM exporter metrics |

Container metrics are collected from the host — no PCP installation inside containers.

## Alternatives Considered

### Prometheus exporters

`node_exporter` + NVIDIA DCGM exporter + Prometheus server. This is the most familiar ecosystem but requires running a Prometheus server or remote write target, exporters expose 1000+ metrics (most unneeded), and it's overkill for a rightsizing tool. Better suited for full observability platforms.

### Custom robne-metrics adapter

A custom lightweight agent that collects only the metrics `librobne` needs. This would be tailored to exactly what the engines consume but would reinvent PCP — historical storage, metric collection, archival, rotation, daemon lifecycle. The maintenance burden is entirely on the `roel` team. The "subset of PCP" needed is basically PCP itself.

### Grafana Agent / Alloy

Lightweight collector that can scrape Prometheus exporters and remote-write. Middle ground but still requires a remote write backend. More moving parts than PCP alone.

## Consequences

- **No external dependencies** — PCP is already on RHEL. No containers, no remote write, no external TSB.
- **Single daemon** — `pmcd` + `pmlogger` is the only daemon to run. No Prometheus, no exporters, no agents.
- **Built-in archival** — `pmlogger` handles historical data with configurable retention. No external database needed for metrics storage.
- **Container-aware** — PCP discovers containers from the host and filters metrics by container name. No in-container agents.
- **Future-proof** — `pmdaopenmetrics` can scrape any `/metrics` endpoint, so custom metrics (e.g., application-specific) can be added without modifying `roel`.

## References

- [PCP documentation](https://pcp.readthedocs.io/en/latest/)
- [PCP container analysis](https://pcp.readthedocs.io/en/latest/QG/AnalyzeLinuxContainers.html)
- [PCP installation](https://pcp.readthedocs.io/en/latest/HowTos/installation/index.html)
- [roel design document](../design/roel-design.md) — Section 4.1: Metrics Collection
