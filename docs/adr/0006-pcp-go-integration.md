# ADR-0006: PCP Go integration via pmproxy REST API

## Status

Accepted

## Context

`roel` is implemented in Go (reusing `librobne` from `ros-ocp-backend`). PCP does not have official Go bindings — the PCP documentation lists C, C++, Perl, and Python as the supported language interfaces. Several options were considered for accessing PCP data from Go.

## Decision

Use the **pmproxy REST API** to access PCP data from Go. `pmproxy` is a standard PCP daemon that exposes PCP metrics over HTTP/JSON. It runs alongside `pmcd` on the target RHEL node.

The integration architecture:

```
libroel (Go)
├── pcp/
│   ├── client.go       — HTTP client for pmproxy REST API
│   ├── metrics.go      — metric discovery and querying
│   └── digest.go       — computes DigestRow from PCP metrics
├── workload/
│   └── key.go          — WorkloadKey abstraction
├── disk/
│   └── engine.go       — disk IOPS/throughput engine
└── ...
```

The `pcp/client.go` is a thin HTTP client:

```go
type Client struct {
    baseURL    string
    httpClient *http.Client
}

func (c *Client) FetchMetrics(ctx context.Context, names []string) ([]Metric, error) {
    // GET /metrics?names=kernel.all.cpu.user,mem.util.used
}

func (c *Client) QuerySeries(ctx context.Context, query string) ([]Sample, error) {
    // POST /series/query with pmseries expression
}
```

## Alternatives Considered

### CGO + libpcp (PMAPI)

PCP's core library (`libpcp`) exposes the PMAPI in C. Calling it from Go via CGO:

```go
/*
#cgo LDFLAGS: -lpcp
#include <pcp/pmapi.h>
*/
import "C"
```

**Rejected because:**
- CGO breaks the `CGO_ENABLED=0` build pattern used by `robne`
- Cross-compilation becomes difficult (can't easily build for amd64 from arm64)
- Ties the build to a specific PCP version and build environment
- Memory management across the C/Go boundary is error-prone
- Conflicts with the goal of a pure Go codebase

### PCP CLI tools + parsing

Shell out to PCP command-line tools and parse their output:

```go
out, _ := exec.Command("pminfo", "--fetch", "kernel.all.cpu.user").Output()
```

**Rejected because:**
- Parsing text output is fragile
- Doesn't scale well for batch queries
- Process spawning overhead
- Hard to handle errors robustly

### pmseries REST API

PCP's time-series query language (`pmseries`) with a REST API backed by Valkey/Redis.

**Rejected because:**
- Requires Valkey/Redis infrastructure
- Overkill for single-node rightsizing
- More moving parts to deploy and maintain

### Python with PCP bindings

PCP has excellent Python bindings. If `roel` were Python, PCP integration would be trivial.

**Rejected because:**
- `librobne` is in Go — would need rewriting or subprocess calls
- The existing ecosystem (CI, linting, deployment) is Go
- Go's performance is better for percentile computation on large datasets

## Consequences

- **Pure Go** — no CGO, consistent with `robne`'s `CGO_ENABLED=0` pattern
- **Clean architecture** — `pmproxy` is a standard PCP component, easy to deploy
- **Sufficient performance** — HTTP/JSON is fine for batch metric queries (daily/hourly aggregates, not real-time streams)
- **Future-proof** — if PCP adds Go bindings later, the transport layer can be swapped without changing the rest of the code
- **Deployment requirement** — `pmproxy` must be running on target RHEL nodes alongside `pmcd`

## References

- [PCP Web Services documentation](https://pcp.readthedocs.io/en/latest/QG/QuickReferenceGuide.html#web-services)
- [pmproxy man page](https://man7.org/linux/man-pages/man1/pmproxy.1.html)
- [ADR-0002](0002-pcp-data-source.md) — PCP is the metrics data source
- [roel design document](../design/roel-design.md) — Section 4.1: Metrics Collection
