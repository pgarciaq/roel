# Contributing to roel

Thank you for your interest in contributing to **roel** (Resource Optimization for Enterprise Linux)!

Contributions of all kinds are welcome — code, bug reports, documentation, testing, feature ideas, and feedback.

## Ways to contribute

| Contribution | How |
|--------------|-----|
| **Code** | Open a pull request against `main` |
| **Bugs** | [File an issue](https://github.com/pgarciaq/roel/issues) with steps to reproduce, expected behavior, and actual behavior |
| **Documentation** | Open a PR with changes to `docs/` or `README.md` |
| **Testing** | Add unit tests, integration tests, or fuzz tests for existing code |
| **Feature ideas** | [File an issue](https://github.com/pgarciaq/roel/issues) describing the problem and proposed solution |
| **Feedback** | Open a [GitHub Discussion](https://github.com/pgarciaq/roel/discussions) or file an issue |

## What is roel?

`roel` provides resource optimization recommendations for **standalone RHEL machines** — bare-metal processes, Podman containers, quadlets, Docker containers, and cgroups. It uses PCP (Performance Co-Pilot) as the metrics data source and reuses the `librobne` recommendation engines from [ros-ocp-backend](https://github.com/pgarciaq/ros-ocp-backend).

See the [conceptual design document](docs/design/roel-design.md) for architecture and design details.

## Code contributions

### Certificate of Origin

By contributing to this project you agree to the Developer Certificate of Origin (DCO). This document was created by the Linux Kernel community and is a simple statement that you, as a contributor, have the legal right to make the contribution. See the [DCO](DCO) file for details.

### Pull request process

1. Fork the repository and create a branch from `main`
2. Make your changes with clear commit messages
3. Add or update tests for any code changes
4. Ensure tests pass: `go test ./...`
5. Open a pull request against `main`
6. A maintainer will review your PR — please be responsive to feedback

### Commit messages

Use imperative mood, reference issue numbers when applicable:

```
Add disk IOPS/throughput recommendation engine

Implement the disk engine that classifies disk types (NVMe, SSD, HDD)
based on observed IOPS and throughput percentiles from PCP metrics.

Fixes #12
```

### Code style

- Follow standard Go conventions (see [Effective Go](https://go.dev/doc/effective_go))
- Use `gofmt` and `go vet` before submitting
- Keep functions small and focused
- Write clear comments for non-obvious logic

## Bug reports

When filing a bug, please include:

1. **Steps to reproduce** — what you did, what you expected, what happened
2. **Environment** — RHEL version, PCP version, roel version
3. **Logs** — relevant log output (redact sensitive information)
4. **Configuration** — any non-default settings

## Feature requests

When requesting a feature, please describe:

1. **The problem** — what gap or pain point does this address?
2. **Proposed solution** — what should the feature do?
3. **Alternatives considered** — what other approaches did you consider?
4. **Acceptance criteria** — how will we know the feature is complete?

## Code of conduct

Be respectful and constructive. We follow the [Contributor Covenant](https://www.contributor-covenant.org/) code of conduct.

## License

By contributing, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE).
