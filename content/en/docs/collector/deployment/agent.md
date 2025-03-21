---
title: Agent
description:
  Why and how to send signals to collectors and from there to backends
weight: 2
cSpell:ignore: prometheusremotewrite
---

The agent collector deployment pattern consists of applications —
[instrumented][instrumentation] with an OpenTelemetry SDK using [OpenTelemetry
protocol (OTLP)][otlp] — or other collectors (using the OTLP exporter) that send
telemetry signals to a [collector][] instance running with the application or on
the same host as the application (such as a sidecar or a daemonset).

Each client-side SDK or downstream collector is configured with a collector
location:

![Decentralized collector deployment concept](../../img/otel-agent-sdk.svg)

1. In the app, the SDK is configured to send OTLP data to a collector.
1. The collector is configured to send telemetry data to one or more backends.

## Example

A concrete example of the agent collector deployment pattern could look as
follows: you manually instrument, say, a [Java application to export
metrics][instrument-java-metrics] using the OpenTelemetry Java SDK. In the
context of the app, you would set the `OTEL_METRICS_EXPORTER` to `otlp` (which
is the default value) and configure the [OTLP exporter][otlp-exporter] with the
address of your collector, for example (in Bash or `zsh` shell):

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT=http://collector.example.com:4318
```

The collector serving at `collector.example.com:4318` would then be configured
like so:

{{< tabpane text=true >}} {{% tab Traces %}}

```yaml
receivers:
  otlp: # the OTLP receiver the app is sending traces to
    protocols:
      grpc:

processors:
  batch:

exporters:
  otlp/jaeger: # Jaeger supports OTLP directly
    endpoint: https://jaeger.example.com:4317

service:
  pipelines:
    traces/dev:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
```

{{% /tab %}} {{% tab Metrics %}}

```yaml
receivers:
  otlp: # the OTLP receiver the app is sending metrics to
    protocols:
      grpc:

processors:
  batch:

exporters:
  prometheusremotewrite: # the PRW exporter, to ingest metrics to backend
    endpoint: https://prw.example.com/v1/api/remote_write

service:
  pipelines:
    metrics/prod:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite]
```

{{% /tab %}} {{% tab Logs %}}

```yaml
receivers:
  otlp: # the OTLP receiver the app is sending logs to
    protocols:
      grpc:

processors:
  batch:

exporters:
  file: # the File Exporter, to ingest logs to local file
    path: ./app42_example.log
    rotation:

service:
  pipelines:
    logs/dev:
      receivers: [otlp]
      processors: [batch]
      exporters: [file]
```

{{% /tab %}} {{< /tabpane >}}

If you want to try it out for yourself, you can have a look at the end-to-end
[Java][java-otlp-example] or [Python][py-otlp-example] examples.

## Tradeoffs

Pros:

- Simple to get started
- Clear 1:1 mapping between application and collector

Cons:

- Scalability (human and load-wise)
- Inflexible

[instrumentation]: /docs/languages/
[otlp]: /docs/specs/otel/protocol/
[collector]: /docs/collector/
[instrument-java-metrics]: /docs/languages/java/instrumentation/#metrics
[otlp-exporter]: /docs/specs/otel/protocol/exporter/
[java-otlp-example]:
  https://github.com/open-telemetry/opentelemetry-java-docs/tree/main/otlp
[py-otlp-example]:
  https://opentelemetry-python.readthedocs.io/en/stable/examples/metrics/instruments/README.html
