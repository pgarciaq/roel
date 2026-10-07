# ADR-0005: Disk IOPS/throughput is a new engine not in librobne

## Status

Accepted

## Context

`librobne` does not have a disk recommendation engine. It has engines for CPU/memory (container, namespace), nodes, GPUs, PVCs, VMs, quotas, and snapshots — but no disk IOPS/throughput recommendations.

Standalone RHEL machines have physical disks (NVMe, SATA SSD, HDD) with different IOPS/throughput characteristics. Rightsizing a disk means recommending the right disk type based on observed I/O patterns — a bare-metal concern that doesn't exist in OpenShift/K8s (where storage is abstracted by PVCs and StorageClasses).

## Decision

`libroel` adds a new `disk` engine that recommends disk types based on observed IOPS and throughput patterns from PCP.

**Input:** Disk I/O metrics from PCP:
- `disk.dev.read_bytes` — bytes read per interval
- `disk.dev.write_bytes` — bytes written per interval
- `disk.dev.read` — read operations per interval
- `disk.dev.write` — write operations per interval
- `disk.dev.read_latency` — read latency
- `disk.dev.write_latency` — write latency

**Output:** Disk type recommendation:

| Observed Pattern | Recommendation | Rationale |
|-----------------|----------------|-----------|
| High IOPS (>10K), low latency | NVMe SSD | High-performance workload |
| Medium IOPS (1K-10K), medium latency | SATA SSD | Balanced workload |
| Low IOPS (<1K), high latency | HDD | Cold storage / archival |
| Very low IOPS, mostly idle | Smaller disk or remove | Over-provisioned |

The engine computes daily percentiles of IOPS and throughput, then classifies the disk based on the observed patterns. It uses the same percentile-based approach as the other `librobne` engines.

## Alternatives Considered

### Reuse librobne's PVC engine

`librobne` has a PVC engine for storage rightsizing, but it recommends PVC size (GiB) based on usage growth — not disk type (NVMe/SSD/HDD) based on IOPS/throughput. These are fundamentally different problems. The PVC engine operates on Kubernetes PVC metrics; the disk engine operates on physical disk I/O metrics.

### No disk engine

Don't recommend disk types — only recommend CPU/memory/GPU rightsizing. This would leave a significant rightsizing gap for standalone RHEL machines, where disk I/O is often the bottleneck.

### Upstream the disk engine to librobne

Add the disk engine to `librobne` so both `ros-ocp-backend` and `roel` can use it. This is a good long-term goal, but the disk engine is RHEL-specific (physical disk types don't exist in OpenShift/K8s). It belongs in `libroel` for now. If a generic version emerges later, it can be upstreamed.

## Consequences

- **New engine** — `libroel/disk` is a new package that doesn't exist in `librobne`.
- **RHEL-specific** — The disk engine recommends physical disk types, which is a bare-metal concern. It is not applicable to OpenShift/K8s.
- **PCP dependency** — The engine reads disk I/O metrics from PCP. It cannot be used without PCP.
- **Percentile-based** — The engine uses the same percentile-based approach as other `librobne` engines, ensuring consistency in the recommendation methodology.

## References

- [roel design document](../design/roel-design.md) — Section 8: Disk IOPS/Throughput Engine
- [librobne engines](https://github.com/pgarciaq/ros-ocp-backend/tree/main/librobne)
- [PCP disk metrics](https://pcp.readthedocs.io/en/latest/QG/QuickReferenceGuide.html)
