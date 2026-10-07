# roel: Resource Optimization for Enterprise Linux

## Conceptual Design Document

**Status:** Draft
**Date:** 2026-10-07

---

## 1. Problem Statement

`ros-ocp-backend` provides resource optimization for OpenShift clusters. It ingests metrics from the koku-metrics-operator (via CSV files uploaded to S3), computes daily percentiles, and produces rightsizing recommendations using the `librobne` engine library.

But many RHEL customers run **standalone RHEL machines** — not OpenShift, not Kubernetes, not OpenShift Virtualization. These machines run bare-metal processes, Podman containers, quadlets, and Docker containers. They have the same rightsizing needs (right-size CPU/memory, right-size disks, optimize GPU utilization) but a completely different data source.

`roel` fills this gap: **resource optimization for standalone RHEL machines**, using PCP (Performance Co-Pilot) as the metrics data source.

---

## 2. Goals and Non-Goals

### Goals

- Provide rightsizing recommendations for standalone RHEL machines based on historical usage metrics
- Reuse the proven `librobne` recommendation engines (node, GPU, container, digest, savings)
- Use PCP as the metrics data source — the RHEL-native monitoring stack
- Support all RHEL workload types: bare-metal processes, Podman containers, quadlets, Docker containers, cgroups
- Provide both a CLI (`roel`) and a daemon (`roeld`) for different consumption patterns
- Follow the same architectural patterns as `ros-ocp-backend` (ingestion → digest → recommend → output)

### Non-Goals

- **Not** a replacement for `ros-ocp-backend` — OpenShift clusters continue to use the existing service
- **Not** a Kubernetes rightsizing tool — that is a separate effort (third-party K8s support)
- **Not** a subscription management tool — RHEL subscription optimization is a different domain
- **Not** a real-time monitoring tool — PCP handles real-time; `roel` focuses on historical rightsizing
- **Not** a replacement for PCP — `roel` consumes PCP metrics, it does not duplicate PCP's collection or storage

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RHEL Node                                   │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  pmcd    │  │ pmlogger │  │ pmdaproc │  │ pmdapodman│           │
│  │ (daemon) │  │ (archive)│  │(process) │  │(podman)   │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │              │                 │
│       └──────────────┴──────────────┴──────────────┘                 │
│                          │                                          │
│                    ┌─────┴─────┐                                    │
│                    │  PMAPI    │                                    │
│                    └─────┬─────┘                                    │
└──────────────────────────┼──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     roel-metrics-adapter                            │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ PCP Client   │→│ Digest       │→│ DigestRow     │             │
│  │ (PMAPI)      │  │ Computer     │  │ Builder      │             │
│  └──────────────┘  └──────────────┘  └──────┬───────┘             │
│                                             │                      │
└─────────────────────────────────────────────┼──────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         libroel                                     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ librobne engines (reused)                                    │  │
│  │  • node.RecommendNodes()                                     │  │
│  │  • gpu.RecommendGPUWithSettings()                            │  │
│  │  • container.RecommendCPUAndMemory()                          │  │
│  │  • digest.ComputeDigest()                                    │  │
│  │  • savings.EstimateSavings()                                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ libroel-specific engines (new)                                │  │
│  │  • disk.RecommendDisks()  — IOPS/throughput-based disk type  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ RHEL workload abstraction                                     │  │
│  │  • Process, Container, Quadlet, Cgroup                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         roel / roeld                                │
│                                                                     │
│  ┌──────────────┐              ┌──────────────┐                    │
│  │ roel (CLI)   │              │ roeld (daemon)│                    │
│  │              │              │              │                     │
│  │ One-shot:    │              │ Long-running:│                    │
│  │ load→digest→ │              │ HTTP API +   │                    │
│  │ recommend→   │              │ scheduled    │                    │
│  │ output       │              │ ingestion    │                    │
│  └──────────────┘              └──────────────┘                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Flow

### 4.1 Metrics Collection (PCP)

PCP runs on each RHEL node with `pcp-zeroconf` (zero-configuration setup). The following PMDAs provide metrics:

| PMDA | Metrics | Used for |
|------|---------|----------|
| `pmdalinux` | `kernel.all.cpu.*`, `mem.util.*`, `disk.dev.*`, `net.dev.*` | System-level CPU, memory, disk IOPS/throughput, network |
| `pmdaproc` | `proc.psinfo.*` | Per-process CPU, RSS, command name |
| `pmdapodman` | Podman container metrics | Container-level CPU, memory, I/O |
| `pmdadocker` | Docker container metrics | Container-level CPU, memory, I/O |
| `pmdasystemd` | `systemd.unit.*` | Quadlet service units |
| `pmdanvidia` | `nvidia.*` | GPU utilization, memory, MIG status |
| `pmdaopenmetrics` | Scrapes `/metrics` endpoints | Custom application metrics, NVIDIA DCGM exporter |

Container metrics are collected **from the host** — no PCP installation inside containers. The `--container <name>` flag filters metrics to a specific container.

### 4.2 Digest Computation

The `roel-metrics-adapter` reads from PCP via PMAPI and computes daily percentiles:

```
For each workload (process, container, quadlet):
  For each day:
    Collect CPU usage samples → compute P50, P60, P95, P98, P99
    Collect memory usage samples → compute P50, P60, P95, P98, P99
    Collect disk I/O samples → compute IOPS/throughput percentiles
    Collect GPU utilization samples → compute percentiles
    → Emit DigestRow
```

The digest computation reuses `librobne/digest` for the percentile math.

### 4.3 Recommendation

`libroel` feeds `DigestRow` to the `librobne` engines:

| Engine | Input | Output |
|--------|-------|--------|
| `node.RecommendNodes()` | Node-level digest | Node size recommendation, consolidation opportunities |
| `gpu.RecommendGPUWithSettings()` | GPU digest | MIG profile, time-slicing recommendation |
| `container.RecommendCPUAndMemory()` | Container/process digest | CPU/memory request/limit recommendation |
| `disk.RecommendDisks()` | Disk IOPS/throughput digest | Disk type recommendation (NVMe, SSD, HDD) |
| `savings.EstimateSavings()` | All recommendations | Cost savings estimate |

### 4.4 Output

**CLI (`roel`):**
- One-shot execution: load PCP data → compute digests → generate recommendations → output JSON/CSV/table
- Suitable for manual runs, CI/CD pipelines, and testing

**Daemon (`roeld`):**
- Long-running service with HTTP API
- Scheduled ingestion and recommendation runs
- Suitable for continuous monitoring and integration with dashboards

---

## 5. Relationship to Existing Projects

### 5.1 `librobne` (reused)

`librobne` is the core recommendation engine library. `libroel` depends on it and reuses:

- `digest/` — percentile computation
- `container/` — CPU/memory sizing algorithm
- `node/` — node sizing and consolidation
- `gpu/` — GPU classification and MIG/time-slicing
- `savings/` — cost estimation
- `fixedpoint/` — basis-point arithmetic
- `types/` — core types (adapted for RHEL)

See [ADR-0001](adr/0001-depend-on-librobne.md) for the dependency decision.

### 5.2 `robne` (sibling CLI)

`robne` is the CLI for `ros-ocp-backend`. `roel` is the CLI for `roel`. They share the same orchestration pattern (load → digest → recommend → output) but have different data sources and domain models.

### 5.3 `ros-ocp-backend` (sibling service)

`ros-ocp-backend` is the OpenShift rightsizing service. `roel` is the RHEL rightsizing service. They share the `librobne` engine library but have completely different ingestion layers, APIs, and domain models.

### 5.4 PCP (data source)

PCP is the metrics data source for `roel`. It is not a sibling project — it is a dependency. `roel` consumes PCP metrics via PMAPI but does not modify or extend PCP.

---

## 6. Key Design Decisions

| Decision | ADR | Status |
|----------|-----|--------|
| `libroel` depends on `librobne` (not independent) | [ADR-0001](adr/0001-depend-on-librobne.md) | Accepted |
| PCP is the metrics data source | [ADR-0002](adr/0002-pcp-data-source.md) | Accepted |
| `roeld` is a daemon (not a cron job or one-shot) | [ADR-0003](adr/0003-roeld-daemon.md) | Accepted |
| RHEL workload abstraction (process, container, quadlet, cgroup) | [ADR-0004](adr/0004-rhel-workload-abstraction.md) | Accepted |
| Disk IOPS/throughput is a new engine not in `librobne` | [ADR-0005](adr/0005-disk-engine.md) | Accepted |

---

## 7. RHEL Workload Abstraction

Standalone RHEL machines have different workload types than OpenShift:

| Workload Type | PCP Source | Description |
|---------------|------------|-------------|
| **Process** | `pmdaproc` | Bare-metal process running directly on RHEL |
| **Podman container** | `pmdapodman` | Container run by Podman |
| **Quadlet** | `pmdasystemd` + `pmdapodman` | Systemd unit managing a Podman container |
| **Docker container** | `pmdadocker` | Container run by Docker |
| **Cgroup** | `cgroup.*` | Any cgroup (processes, containers, services) |

Each workload type maps to a `WorkloadKey` in `libroel`:

```go
type WorkloadKey struct {
    Type        WorkloadType  // process, container, quadlet, cgroup
    Name        string        // process name, container name, quadlet name
    PID         int           // for processes
    ContainerID string        // for containers
    CgroupPath  string        // for cgroups
}
```

This is distinct from `librobne`'s `ContainerKey` (which has `Namespace`, `Workload`, `WorkloadType`, `ContainerName` — all Kubernetes concepts).

See [ADR-0004](adr/0004-rhel-workload-abstraction.md) for the full workload abstraction design.

---

## 8. Disk IOPS/Throughput Engine

`librobne` does not have a disk recommendation engine. `libroel` adds one:

**Input:** Disk IOPS and throughput metrics from PCP (`disk.dev.read_bytes`, `disk.dev.write_bytes`, `disk.dev.read`, `disk.dev.write`)

**Output:** Disk type recommendation based on observed IOPS/throughput patterns:

| Observed Pattern | Recommendation |
|-----------------|----------------|
| High IOPS (>10K), low latency | NVMe SSD |
| Medium IOPS (1K-10K), medium latency | SATA SSD |
| Low IOPS (<1K), high latency | HDD |
| Very low IOPS, mostly idle | Consider smaller disk or remove |

This engine is RHEL-specific because it recommends physical disk types, which is a bare-metal concern. In OpenShift/K8s, storage is abstracted away by PVCs and StorageClasses.

See [ADR-0005](adr/0005-disk-engine.md) for the full disk engine design.

---

## 9. Module Structure

```
roel/
├── docs/
│   ├── design/
│   │   └── roel-design.md          # This document
│   └── adr/
│       ├── 0001-depend-on-librobne.md
│       ├── 0002-pcp-data-source.md
│       ├── 0003-roeld-daemon.md
│       ├── 0004-rhel-workload-abstraction.md
│       └── 0005-disk-engine.md
├── libroel/                        # Go module
│   ├── go.mod                      # module github.com/pgarciaq/roel/libroel
│   ├── pcp/                        # PCP client and metric discovery
│   ├── digest/                     # Digest computation (wraps librobne/digest)
│   ├── workload/                   # RHEL workload abstraction
│   ├── disk/                       # Disk IOPS/throughput engine
│   ├── config/                     # RHEL-specific configs and rate cards
│   └── types/                      # RHEL-specific types (WorkloadKey, etc.)
├── cmd/
│   └── roel/                       # CLI
│       └── main.go
└── internal/
    └── roeld/                     # Daemon
        ├── server.go               # HTTP API
        ├── ingestion.go            # Scheduled ingestion
        └── recommend.go            # Recommendation runner
```

---

## 10. Open Questions

These are not yet settled and will be addressed as the design evolves:

1. **How does `roeld` discover RHEL nodes?** Manual configuration? PCP's discovery? Some other mechanism?
2. **What is the retention policy for PCP archives?** PCP's `pmlogger` has its own retention; does `roel` add another layer?
3. **How does `roel` handle RHEL subscription data?** Does it need to know about RHEL subscriptions to estimate savings?
4. **What is the API surface for `roeld`?** REST? gRPC? What endpoints?
5. **How does `roel` handle multiple RHEL nodes?** Is it per-node or fleet-level?
6. **Does `roel` need a database?** Or is it stateless, reading directly from PCP archives?

---

## 11. References

- [ros-ocp-backend](https://github.com/pgarciaq/ros-ocp-backend) — OpenShift rightsizing service
- [librobne](https://github.com/pgarciaq/ros-ocp-backend/tree/main/librobne) — Recommendation engine library
- [robne](https://github.com/pgarciaq/ros-ocp-backend/tree/main/cmd/robne) — CLI for ros-ocp-backend
- [PCP documentation](https://pcp.readthedocs.io/en/latest/) — Performance Co-Pilot
- [PCP container analysis](https://pcp.readthedocs.io/en/latest/QG/AnalyzeLinuxContainers.html) — Container metrics in PCP
