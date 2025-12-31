# OpenTelemetry Collector + Google Cloud Metrics Exporter: Duplicate TimeSeries Issue

## Context

When using the OpenTelemetry Collector with the Google Cloud Monitoring (GCM) exporter for high-frequency metrics, you may encounter the following error:

```
Duplicate TimeSeries encountered. Only one point can be written per TimeSeries per request.
```

This happens when multiple points for the **same metric + label set + resource** are sent in a single export request.

---

## Example Log from the POC of this repo (Prettyfied)
```
2025-12-31T18:03:23.607Z    ERROR
otelcol.component.id: googlecloud
otelcol.component.kind: exporter
otelcol.signal: metrics

Resource:
  service.name: otelcol-contrib
  service.instance.id: 5a9e60d0-1afa-43ec-ae28-275e79b564cf
  service.version: 0.142.0

Error:
  Failed to export metrics to projects/alfred-eyes-staging
  rpc error: code = InvalidArgument
  desc = One or more TimeSeries could not be written

Duplicate TimeSeries details:
  metric.type: "custom.googleapis.com/otel-poc-timeseries-bug/mock_rando"
  resource.type: "generic_node"
  resource.labels: { location: "global", node_id: "", namespace: "" }

  timeSeries[1]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[2]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[3]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[4]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[5]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[6]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[7]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[8]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[9]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

  timeSeries[10]:
    metric.labels: { service_name: "poc-otel-collector-timeseries-bug", host: "727c1458b3cb" }

Error summary:
  total_point_count: 10
  success_point_count: 1
  errors:
    - status.code: 3
      point_count: 9
  dropped_items: 10
```

---

## Root Cause

1. **GCM API constraint**
   - `CreateTimeSeries` requests **only allow one point per TimeSeries per request**.
   - A `TimeSeries` can store multiple points historically, but ingestion API enforces **one point per request**.

2. **OTel Collector batching**
   - The Collector batches metrics for efficiency, **regardless of downstream API semantics**.
   - Multiple points for the same TimeSeries in one batch trigger the Duplicate TimeSeries error.

3. **Semantic mismatch**
   - OTel batching semantics: allow multiple points per request.  
   - GCM ingestion semantics: one point per TimeSeries per request.

---

## Remove Batching?

Removing batching entirely would avoid the error but introduces:

- High network overhead (hundreds/thousands of requests per second)
- Higher CPU/memory usage in Collector
- Increased risk of partial failures

## Resolution Path / Workarounds

To respect GCM constraints while keeping batching:

1. Reduce batch timeout
   - Example: `batch.timeout: 1s`  
   - Ensures only one point per TimeSeries per batch.

2. Reduce metric frequency
   - Align the generation frequency with batch size / timeout.

3. Aggregate multiple points
   - Sum, average, min/max multiple high-frequency points -> one point per batch.

4. Generate unique TimeSeries
   - Add a dynamic label or attribute per point to create distinct TimeSeries.
