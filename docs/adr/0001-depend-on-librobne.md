# ADR-0001: libroel depends on librobne (not independent)

## Status

Accepted

## Context

`librobne` is the recommendation engine library extracted from `ros-ocp-backend`. It contains the core algorithms for rightsizing: percentile computation, decay weighting, adaptive margin, node sizing, GPU classification, cost estimation, and more. These algorithms are platform-agnostic — they operate on usage metrics and produce sizing recommendations regardless of whether the workload runs on OpenShift, standalone RHEL, or any other platform.

`roel` needs the same recommendation math. The question is whether to depend on `librobne` or create an independent `libroel` library.

## Decision

`libroel` depends on `librobne`. It imports `librobne` and reuses its engines directly. `libroel` adds RHEL-specific functionality on top: PCP ingestion adapter, RHEL workload abstraction, disk IOPS/throughput engine, and RHEL-specific configurations.

The dependency graph:

```
libroel (Go module)
├── imports librobne → engines, digest, savings, fixedpoint
├── pcp/ → PMAPI client, metric discovery, DigestRow builder
├── workload/ → RHEL workload abstraction (process, container, quadlet, cgroup)
├── disk/ → NEW: disk IOPS/throughput recommendation engine
├── config/ → RHEL-specific configs and rate cards
└── types/ → RHEL-specific types (WorkloadKey, etc.)
```

## Alternatives Considered

### Fully independent libroel

Copy or fork the generic parts of `librobne` into `libroel`. This would avoid the dependency but result in code duplication, double maintenance of bug fixes, and two test suites. The "benefit" of full independence is illusory — `librobne` is already a separate Go module with a clean API, so depending on it is no different from depending on any other library.

### Extract a shared core library (libropt)

Refactor `librobne` to extract a truly generic core (`libropt`), then have both `librobne` and `libroel` depend on it. This is the cleanest architecture but requires refactoring `librobne` now, which is premature. If the OpenShift naming in `librobne`'s types becomes painful later, this refactor can be done at that time. The Go module system makes this straightforward.

## Consequences

- **Code reuse:** Thousands of lines of well-tested recommendation logic are reused directly.
- **Bug fixes:** A fix in `librobne`'s percentile math benefits both `ros-ocp-backend` and `roel`.
- **Coupling:** `libroel` is coupled to `librobne`'s API. If `librobne`'s API changes, `libroel` must adapt. This is mitigated by `librobne` being a stable, versioned Go module.
- **OpenShift naming:** `librobne`'s types have OpenShift-specific names (`ContainerKey`, `NamespaceRow`, `DigestRow` with `PodCountMin/Max`). `libroel` defines its own types (`WorkloadKey`, `WorkloadDigest`) and converts to `librobne`'s types at the engine boundary. This keeps the RHEL domain model clean.

## References

- [librobne](https://github.com/pgarciaq/ros-ocp-backend/tree/main/librobne) — Recommendation engine library
- [ADR-0303](https://github.com/pgarciaq/ros-ocp-backend/blob/main/docs/adr/0303-library-extraction-librobne.md) — Library extraction of librobne
- [roel design document](../design/roel-design.md) — Section 5: Relationship to existing projects
