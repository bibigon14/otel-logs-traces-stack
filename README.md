# otel-logs-traces-stack

[![validate](https://github.com/bibigon14/otel-logs-traces-stack/actions/workflows/validate.yml/badge.svg)](https://github.com/bibigon14/otel-logs-traces-stack/actions/workflows/validate.yml)

Logs and traces half of the observability triad, dropped onto the same
Oracle Cloud Ampere A1 free-tier VM that already runs Prometheus and
VictoriaMetrics for metrics (see [vm-observability](https://github.com/bibigon14/vm-observability)).
The result is one Grafana pane with metrics, logs and traces all
pointing at the same timestamp and the same pod, built from four
upstream pieces glued together:

- **Grafana Loki** - log aggregation, 7d retention on local PVC
- **Grafana Tempo** - trace storage, 72h retention on local PVC
- **OpenTelemetry Collector** - gateway + daemonset agent, OTLP in,
  Loki + Tempo + Prometheus-remote-write out
- **Grafana Alloy** - replacement for Promtail, scrapes pod stdout
  and forwards to Loki over OTLP

## Why this project

Every observability demo online uses `docker-compose up` and three sample
services. This one runs on a real multi-tenant single-node k3s cluster
alongside two production-shape metric stacks, a Kafka cluster with a
JVM producer/consumer workload, Cilium with Hubble L7, and an eBPF
TCP agent. The resource budget is honest, the scrape surface is real,
and the correlation story is the point - the demo isn't "it works",
the demo is "when the Kafka consumer lag SLO burns, you click the
alert, land in Grafana, and already have the trace span and the error
log line waiting."

## What's in the box (v0.1)

- Loki single-binary mode with filesystem backend (BoltDB index + chunks on PVC)
- Tempo single-binary mode with filesystem backend
- OTEL Collector Deployment (gateway) + DaemonSet (host agent)
- Alloy DaemonSet for pod log collection
- Grafana datasource ConfigMaps for Loki and Tempo (sidecar-provisioned)
- A unified dashboard that overlays metrics / logs / traces on one time
  axis for the Kafka demo workload
- Example instrumented Go service (`demo-trace-emitter`) that emits
  structured JSON logs and OTLP traces, so the stack has traffic to show

## What's deliberately out of scope for v0.1

- **Loki in simple-scalable or microservices mode** - single-binary is
  fine for a 2 OCPU / 12 GiB host and the migration path is well
  documented upstream; v0.3 if the cardinality ever justifies it
- **Tempo metrics generator** - would need Prometheus remote-write
  back into the metric stack, out of scope for the first slice
- **Object storage backend** (S3 / R2) - keeps the repo self-contained;
  file backend is the right default for a home-scale deployment
- **Log retention tiering** - 7 days is enough for a demo; a real
  production setup would push to S3 and compact

## Architecture

See [docs/architecture.md](docs/architecture.md) for the full topology
diagram, the OTEL pipeline shape, and the correlation story.

## Resource budget on OCI Ampere A1

See [docs/resource-budget.md](docs/resource-budget.md) for measured RSS
of every pod in the stack and the headroom analysis against the shared
2 OCPU / 12 GiB host.

## Install

```bash
# Install the OpenTelemetry Operator (CRDs for OpenTelemetryCollector + Instrumentation)
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# Create the namespace + core workloads
kubectl apply -f manifests/

# Expose Grafana (if not already port-forwarded)
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```

Open `http://localhost:3000`, go to Explore, pick the `Loki` or `Tempo`
datasource. The Kafka demo dashboard (`kafka-tri-signal`) overlays all
three signals.

## Related

- [vm-observability](https://github.com/bibigon14/vm-observability) - metrics half of the triad
- [kafka-slo-observability](https://github.com/bibigon14/kafka-slo-observability) - the workload we're correlating against
- [cilium-l7-observability](https://github.com/bibigon14/cilium-l7-observability) - L7 flow data on the same cluster
