# ADR-0004: RHEL workload abstraction (process, container, quadlet, cgroup)

## Status

Accepted

## Context

Standalone RHEL machines have different workload types than OpenShift. OpenShift has pods, namespaces, deployments, and machinesets. Standalone RHEL has:

- **Processes** — bare-metal processes running directly on RHEL
- **Podman containers** — containers run by Podman
- **Quadlets** — systemd units that manage Podman containers
- **Docker containers** — containers run by Docker
- **Cgroups** — any cgroup (processes, containers, services)

`librobne`'s `ContainerKey` is Kubernetes-specific (`Namespace`, `Workload`, `WorkloadType`, `ContainerName`). `roel` needs its own workload abstraction that maps to RHEL concepts.

## Decision

`libroel` defines its own `WorkloadKey` type that abstracts RHEL workloads:

```go
type WorkloadType string

const (
    WorkloadTypeProcess   WorkloadType = "process"
    WorkloadTypeContainer WorkloadType = "container"
    WorkloadTypeQuadlet   WorkloadType = "quadlet"
    WorkloadTypeCgroup    WorkloadType = "cgroup"
)

type WorkloadKey struct {
    Type        WorkloadType
    Name        string  // process name, container name, quadlet name
    PID         int     // for processes
    ContainerID string  // for containers
    CgroupPath  string  // for cgroups
}
```

Each workload type maps to PCP metrics:

| Workload Type | PCP Source | Key Fields |
|---------------|------------|------------|
| Process | `pmdaproc` | PID, command name |
| Podman container | `pmdapodman` | Container ID, name |
| Quadlet | `pmdasystemd` + `pmdapodman` | Unit name, container name |
| Docker container | `pmdadocker` | Container ID, name |
| Cgroup | `cgroup.*` | Cgroup path |

`libroel` converts `WorkloadKey` to `librobne`'s `ContainerKey` at the engine boundary. The conversion is a simple struct mapping — the underlying recommendation algorithms don't care about the workload type.

## Alternatives Considered

### Reuse librobne's ContainerKey as-is

Use `librobne`'s `ContainerKey` directly, ignoring the Kubernetes-specific fields. This would avoid a conversion layer but would leak OpenShift naming into the RHEL domain model. Code would have `Namespace` and `WorkloadType` fields that don't make sense for standalone RHEL.

### Define a generic WorkloadKey in librobne

Add a generic `WorkloadKey` to `librobne` and have both `ros-ocp-backend` and `roel` use it. This would be the cleanest long-term solution but requires modifying `librobne` now, which is premature. If the OpenShift naming becomes painful, this refactor can be done later.

### Define WorkloadKey in libroel (chosen)

Define `WorkloadKey` in `libroel` and convert to `librobne`'s types at the engine boundary. This keeps the RHEL domain model clean without modifying `librobne`. The conversion is a simple struct mapping.

## Consequences

- **Clean RHEL domain model** — `roel` code uses `WorkloadKey` with RHEL concepts, not OpenShift concepts.
- **Conversion layer** — a thin conversion from `WorkloadKey` to `ContainerKey` at the engine boundary. This is a simple struct mapping, not a complex transformation.
- **Extensibility** — new workload types can be added to `WorkloadKey` without modifying `librobne`.
- **Future refactor path** — if `librobne` later adds a generic `WorkloadKey`, the conversion layer can be removed with minimal changes.

## References

- [roel design document](../design/roel-design.md) — Section 7: RHEL Workload Abstraction
- [PCP container analysis](https://pcp.readthedocs.io/en/latest/QG/AnalyzeLinuxContainers.html)
- [librobne types](https://github.com/pgarciaq/ros-ocp-backend/tree/main/librobne/types)
