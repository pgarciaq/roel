# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for roel.
Each record captures a significant architectural decision, its context, and consequences.

Format follows [Michael Nygard's ADR template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

## Index

| Number | Title | Domain | Status |
|--------|-------|--------|--------|
| [0001](0001-depend-on-librobne.md) | libroel depends on librobne (not independent) | Architecture | Accepted |
| [0002](0002-pcp-data-source.md) | PCP is the metrics data source | Data Source | Accepted |
| [0003](0003-roeld-daemon.md) | roeld is a daemon (not a cron job or one-shot) | Deployment | Accepted |
| [0004](0004-rhel-workload-abstraction.md) | RHEL workload abstraction (process, container, quadlet, cgroup) | Domain Model | Accepted |
| [0005](0005-disk-engine.md) | Disk IOPS/throughput is a new engine not in librobne | Engine | Accepted |
