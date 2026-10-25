# Resource budget on OCI Ampere A1

The whole point of running Loki + Tempo + OTEL Collector on a free-tier
VM alongside two other metric stacks is that it has to leave enough
headroom for the actual workloads being observed. The host is a 2 OCPU
/ 12 GiB Ampere A1 instance already carrying kube-prometheus-stack,
vm-observability, a Strimzi Kafka cluster (3 brokers), Cilium CNI, and
an eBPF observer DaemonSet.

## Measured steady-state (v0.1, 2026-10-25)

Taken from `kubectl top pods -n observability` ~15 minutes after
rollout, with telemetrygen pushing 5 traces/s and 2 log lines/s through
the full pipeline:

| Pod | CPU | RSS |
|---|---:|---:|
| loki-0 | 4m | **70 MiB** |
| tempo-0 | 2m | **391 MiB** |
| otelcol-gateway | 5m | **53 MiB** |
| log-emitter | 2m | 8 MiB |
| trace-emitter | 2m | 10 MiB |
| **Total scrape plane** | **15m** | **532 MiB** |

![Live kubectl top on the OCI VM showing 532 MiB total RSS and 2.0 GiB available memory on the shared host](screenshots/04-kubectl-top.png)

Host state in that same screenshot:

- `Mem: 11Gi total, 9.7Gi used, 197Mi free, 2.0Gi buff/cache, 2.0Gi available`
- `Swap: 0B` (as expected for an Ampere A1 free-tier instance)

The 9.7 GiB `used` is almost all the other tenants of this host - Kafka
JVM heap (3 brokers × ~1.3 GiB), kube-prometheus Prometheus pod
(~650 MiB), Grafana (~500 MiB), VMSingle (~300 MiB), node-exporter and
all the operators. The 532 MiB from this stack is a 4.7% add-on to a
10 GiB working set.

## Why Tempo dominates

391 MiB on `tempo-0` is 73% of the stack's RSS, which is the honest
cost of running trace storage in-process rather than as a separate
cluster. The breakdown inside that pod:

- **Ingester WAL buffer** (~150 MiB) - holds in-flight spans before
  they flush to a block. `ingester.max_block_bytes: 1000000` and
  `ingester.max_block_duration: 5m` are already tuned small for this
  host; the default 10x higher ceilings would push RSS past 1 GiB.
- **TSDB-like block cache** (~100 MiB) - compacted blocks held in
  memory for fast query hits.
- **Compactor working set** (~80 MiB) - active compaction holds
  source and destination blocks simultaneously.
- **Go runtime + distributor + querier overhead** (~60 MiB).

Loki by comparison keeps 70 MiB because its `boltdb-shipper` index
mode writes to disk on every flush and queries scan compressed chunks
rather than keeping them in memory.

## What breaks this budget

- **Scrape surface growth**: cluster-wide stdout log collection via
  Alloy would add ~100 MiB (Alloy itself) plus Loki ingest growth
  proportional to log volume. Our ~1200 pods × 10 lines/s would roughly
  double Loki RSS.
- **Trace cardinality**: moving from telemetrygen's single-span flat
  traces to a real instrumented workload with deep span trees would
  push Tempo past 500 MiB. The ingester's `max_live_traces` would need
  tuning.
- **Enabling Tempo's `metrics_generator`**: adds a Prometheus
  remote-write path and ~80 MiB RSS. Planned for v0.2 but intentionally
  off here.

## Comparison against the all-Grafana Helm chart

For reference, Grafana's own `loki-distributed` chart in default
single-replica mode runs ~450 MiB on this host (distributor + ingester
+ querier + query-frontend + gateway, all separate pods). Running
single-binary cuts that roughly 6x at the cost of HA. On a free-tier
VM with no HA requirement anyway, the trade is honest.
