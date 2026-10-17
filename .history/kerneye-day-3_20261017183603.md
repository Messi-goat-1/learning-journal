## Traces page: show agent (eBPF) traces by default

### Why
KernEye should not require customers to change their application code.
The agent already captures traces and spans with eBPF. That is enough to see
where a problem is (which service, which call, how slow, which errors),
without the customer adding OpenTelemetry to their apps.

### Problem
The Traces page was empty, even though the database had hundreds of
thousands of traces.

KernEye has two trace sources, both stored in the same ClickHouse table (`otel_traces`):
- **OpenTelemetry**: sent by the application itself (requires code changes).
- **eBPF**: captured automatically by the agent. `ServiceName` starts with `/`.

On the test server, every trace came from eBPF and none from OpenTelemetry:

```
$ docker exec -it kerneye-clickhouse-1 clickhouse-client -q \
  "SELECT ServiceName, count() FROM otel_traces GROUP BY ServiceName"
/docker/kerneye-server-1              638970
/docker/kerneye-agent-1                44803
/system.slice/k3s.service              45705
/docker/example-voting-app-db-1          886
/docker/example-voting-app-worker-1    80615
/k8s/kube-system/metrics-server         5261
/docker/example-voting-app-result-1     8200
/docker/example-voting-app-vote-1         15
/docker/example-voting-app-redis-1       100
```

### Root cause
`api/views/overview/overview.go:44` calls `RenderTraces(..., false)` for the
Traces page. With `ebpfFallback = false`, the page looks only for OpenTelemetry
traces, finds none, and returns `not_found`.

This was intentional upstream (Coroot), not a bug: `api/mcp.go:1397` calls the
same function with `true`, so the AI/MCP side sees eBPF traces while the UI hides
them, likely because eBPF traces show calls between services, not functions
inside a request.

### Side problem: platform noise
Once eBPF traces are shown, KernEye's own containers dominate the page
(`kerneye-server` alone has 638,970 spans), because the agent traces every
container, including the platform itself.

### Changes
- Traces page and heatmap always read agent (eBPF) traces.
- New filter at the top, so the user picks what to see:
  - **application**: customer apps (on by default)
  - **monitoring (KernEye)**: the platform's own containers (off by default)
  - Saved in the URL as `groups`.
- OpenTelemetry UI hidden behind `OtelTracesUIEnabled` (Go) and
  `otelEnabled` (Vue). No code deleted.
- TRACING tab ignores OpenTelemetry services while the flag is off.
- MCP, old links (`include_aux`) and OTLP ingestion are unchanged.

### Testing
- `go build`, `go vet`, `go test ./...` (2 new tests), frontend lint and build: pass.
- Manually tested on a server with the Docker example voting app.

### Rollback
Set `OtelTracesUIEnabled = true` and `otelEnabled = true`, or revert this PR.
No schema changes.