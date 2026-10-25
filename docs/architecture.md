# Architecture

## Topology

```
                     OCI Ampere A1 VM (shared with vm-observability + kafka-slo)
 +-----------------------------------------------------------------------------+
 |                                 k3s v1.36                                   |
 |                                                                             |
 |  namespace: observability                                                   |
 |                                                                             |
 |  +------------------+   OTLP   +---------------------+   OTLP   +--------+  |
 |  | Alloy (DS)       |--------->| OTEL Collector      |--------->| Tempo  |  |
 |  | tails pod stdout |          | (Deployment,        |          | (STS,  |  |
 |  +------------------+          |  gateway role)      |          |  file) |  |
 |                                |                     |   loki   +--------+  |
 |                                |                     | pushapi               |
 |                                |                     +---------->+-------+   |
 |                                +---------------------+           | Loki  |   |
 |                                   ^      ^       ^               | (STS, |   |
 |                                   |      |       |               |  file)|   |
 |     +-----------+                 |      |       |               +-------+   |
 |     | demo-     |----OTLP traces--+      |       |                           |
 |     | trace-    |----OTLP logs-----------+       |                           |
 |     | emitter   |----OTLP metrics----------------+                           |
 |     +-----------+                                     (metrics forwarded     |
 |                                                        to VMSingle via       |
 |                                                        prometheusremotewrite)|
 |                                                                             |
 |  namespace: monitoring (existing, from kube-prometheus-stack)               |
 |  +----------------------------+                                             |
 |  | Grafana + sidecars         |---query--+---> Loki                          |
 |  | datasources: Prom, VM,     |          +---> Tempo                         |
 |  | Loki (new), Tempo (new)    |          +---> Prom (metrics)                |
 |  +----------------------------+          +---> VMSingle (metrics)            |
 |                                                                             |
 +-----------------------------------------------------------------------------+
```

## OTEL Collector pipeline shape

The gateway collector runs one pipeline per signal (traces, logs,
metrics). The DaemonSet agent is thinner - it only receives host and
pod telemetry and forwards everything over OTLP to the gateway, so
the pipeline configuration lives in one place.

```
receivers:
  otlp:                       # OTLP from Alloy, apps, agent collectors
    protocols: {grpc, http}
  hostmetrics:                # on the DaemonSet only
    collection_interval: 30s

processors:
  memory_limiter:             # protect the collector pod from OOM
    limit_mib: 300
  batch:                      # batch for exporter efficiency
    timeout: 5s
    send_batch_size: 1024
  k8sattributes:              # enrich with pod/ns/node labels
    passthrough: false
    extract:
      metadata: [k8s.pod.name, k8s.namespace.name, k8s.node.name]
  resource:                   # tag everything with cluster id

exporters:
  otlp/tempo:
    endpoint: tempo.observability.svc:4317
    tls: {insecure: true}
  loki:
    endpoint: http://loki.observability.svc:3100/loki/api/v1/push
  prometheusremotewrite:
    endpoint: http://vmsingle-main.vm.svc:8429/api/v1/write

service:
  pipelines:
    traces:  {receivers: [otlp], processors: [memory_limiter, k8sattributes, batch], exporters: [otlp/tempo]}
    logs:    {receivers: [otlp], processors: [memory_limiter, k8sattributes, batch], exporters: [loki]}
    metrics: {receivers: [otlp, hostmetrics], processors: [memory_limiter, batch], exporters: [prometheusremotewrite]}
```

## The correlation story

The point of running Loki + Tempo + Prometheus against the same cluster
isn't three dashboards - it's one. Grafana's **trace-to-logs** and
**metrics-to-traces** datasource links wire together as follows:

1. **Alert fires**: VMAlert evaluates a burn-rate rule from
   `vm-observability`, say `KafkaConsumerLagBurnRateFast`. The
   Alertmanager webhook includes a trace exemplar ID (vmagent is
   configured with `exemplarsFromMetadata: true` to pass exemplars
   through from the kafka-exporter).
2. **Click through to the dashboard**: Grafana opens the Kafka SLO
   dashboard. Lag panel shows the burn curve with trace-ID dots
   plotted as exemplars.
3. **Click an exemplar**: Grafana follows the metrics-to-traces link
   to Tempo, pivoting on the trace ID. The waterfall shows the
   consumer's `commit` span taking 2.3s against the broker (vs 50ms
   baseline).
4. **Click a span**: trace-to-logs link opens Loki, filtered by
   `trace_id={that trace}` and `namespace="kafka"`. Three log lines
   from the broker pod show the actual rebalance event.

No two tabs, no copy-pasting pod names, no "when did this start"
guesswork. The alert payload carries the correlation key and Grafana
walks the rest.

## Why not include this in the metrics project

`vm-observability` is already busy proving one specific thing - that
VictoriaMetrics draws roughly a third less RSS than Prometheus on the
same scrape surface. Jamming Loki + Tempo in alongside would muddy that
benchmark (shared memory pressure skews the measurement) and would also
force one repo to carry four distinct upstream chart conventions. Two
repos, one shared cluster, independent CI.
