# Project Overview
The **Prometheus Cardinality Exporter** is a specialized tool designed to expose granular TSDB cardinality metrics from Prometheus instances. It identifies high-cardinality labels and series that cause performance degradation by querying the `/api/v1/status/tsdb` endpoint. It features native Kubernetes Service Discovery to automatically monitor sharded Prometheus deployments.

## Tech Stack
- **Language:** Go 1.25.8
- **Metrics:** Prometheus Client Golang v1.23.2
- **Orchestration:** Kubernetes Client-Go v0.35.2
- **CLI Parsing:** go-flags v1.6.1
- **Logging:** Logrus v1.9.4
- **Resiliency:** cenkalti/backoff v4.3.0

## Core Conventions
- **State Management:** Per-instance state is maintained in `cardinalityInfoByInstance` (map). Metrics are updated in-place via Gauges defined in the `cardinality` package.
- **Concurrency:** The main loop in `collectMetrics()` runs in a separate goroutine. Kubernetes API calls use `context.TODO()`.
- **Package Structure:** 
  - `main.go`: Entry point, CLI flag parsing, and Kubernetes service discovery logic.
  - `cardinality/`: Core logic for fetching TSDB status, parsing results, and exposing metrics.
- **Retries:** All external API calls (Prometheus/K8s) MUST use `backoff.v4` for exponential retries to handle transient network issues.

## Build & Test Commands
- **Build Binary:** `go build -o exporter .`
- **Run Tests:** `go test ./...` (Comprehensive unit tests and E2E coverage)
- **Lint Code:** `golangci-lint run`
- **Docker Build (Dev):** `docker build -f Dockerfile-builder . -t cardinality-exporter:dev`
- **Docker Build (Prod):** `docker build -f Dockerfile-builder_distroless . -t cardinality-exporter:latest`

## Rules & Constraints
### Do
- **Update Documentation:** Always update the `AGENTS.md` and `README.md` files after each modification to ensure documentation stays in sync with the codebase.
- **Check Errors:** Always check and handle errors from API calls and parsing logic.
- **Log via Logrus:** Use the global `log` (logrus) instance; never use `fmt.Printf` for application logging.
- **Use Pointers for Instances:** Pass pointers to `PrometheusCardinalityInstance` to avoid copying state.
- **Respect Stats Limit:** Always pass `opts.StatsLimit` to TSDB queries to prevent overloading the target Prometheus.

### Don't
- **Hardcode Credentials:** Never hardcode auth tokens or passwords; use the `--auth` flag mechanism.
- **Direct Metric Manipulation:** Don't manipulate Prometheus Gauges outside of the `cardinality` package logic.
- **Block Main Thread:** Ensure metric collection loops do not block the HTTP server's ability to serve `/health` or `/metrics`.
- **Use v2 Backoff:** Do not use the deprecated `backoff` v2; always use `v4`.
