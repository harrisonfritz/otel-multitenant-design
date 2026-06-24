# OTel Multitenant Design 

## Problem
In massive multi-tenant Kubernetes clusters, the lack of decentralized configuration in the Opentelemetry Collector forces platform teams into an unscalable compromise: 
they must either maintain 
1. a colossal, monoithic configuration that hardcodes the routing logic and destinations for thousands of distinct tenants or relies on complex filtering/routing logic 
2. or deploy hundreds of heavy, resource-intensive Collector instances into individual namespaces to give teams isolation.
3. or give tenants access to edit a centralized collector configuration resulting in a huge centralized configuration, frequent restarts, and issues

## Problem scenarios

### Overall
- Noisy Neighbor problem (some tenants produce or rely on a ton of telemetry)
- Tenants may want to export to different backend destinations (s3, splunk, newrelic, etc.)
- Some tenants don't need any telemetry
- Some tenants may only need a certain telemetry signal (just logs, versus metrics and traces for example)

### Logs/Traces
- Some tenants can be sampled, others cannot
- Some tenants produce structure logs, others do not
- Some tenants rely on adding specific custom metadata (attributes) to their logs 

### Metrics
- High cardinality metrics - it's difficult to know what metrics and attributes tenants need so dropping attributes or metrics that are expensive 

## Current State Solutions

### The Ship Everything Approach
> Ship everything by default and rely on frequent collector updates and complex routing/filtering/sampling to manage volume for tenants. 

**problems:**
- Costs usually explode and lots of unused telemetry is shipped. 
- Some issues/outages are caused by filtering/sampling.
- Collector admin is still making frequent updates that impact many tenants at a time

```
config:
  routing:
    default_pipelines:
    -logs/unsampled
    table:
    - condition: resource.attributes["k8s.namespace.labels.samplelogs"] == true
      context: log
      pipelines:
      - logs/sampled
    - condition: resource.attributes["k8s.namespace.name"] == "tenant-a-special-pipeline"
      context: log
      pipelines:
      - logs/tenant-a-special-pipeline
    - condition: resource.attributes["k8s.namespace.name"] == "tenant-b-special-pipeline"
      context: log
      pipelines:
      - logs/tenant-b-special-pipeline

    filter:
      logs:
        log_record:
        - resource.attributes["k8s.namespace.name"] == "noisy-tenant-c"
```

### Tenants bring their own collector approach
> tenants push all telemetry to their own collector that they run in their namespace. Some teams write their own controllers to update a routing table and route telemetry back to a tenants namespaced collector 

**problems:**
- Creates a ton of unnecessary otel deployments eating up a lot of CPU and memory. 
- Works for metrics/traces, but for logs a daemonset is required
- No centralized collector point of control to sample/block/redact specific namespaces

## Target State Solutions

Create a way for tenants to be fully responsible for their own telemetry without directly editing the centralized configuration. 

The tenant configuration should be as close as possible to the existing opentelemetry collector yaml API

Several other projects have already had success with decentralized configuration:
- [prometheus pod monitors custmo resource]
- [fluentdconfig namespaced custom resource]
- [istio telemetry custom resource]

Tenants should be able to define their own otel pipeline that is injected into the centralized opentelemetry collector configuration
```
```

### Problems with target state
- how do tenants choose which collector injects their pipeline?
- how to protect the centralized collector from misconfigured tenant pipelines or an extremely resource intensive tenant pipeline
- how to prevent tenants from adding a processor or filter that impacts another tenant's namespace?
- tenant pipeline versioning is coupled to collector versioning (if a processor configuration is updated in the new collector version, the tenant pipeline configuration must change to match it.

### Custom Resource example:
```
# =============================================================================
# TenantPipeline — a namespaced custom resource that lets a tenant own their
# OTel pipeline without editing the central collector config.
#
# spec.config is the EXACT same schema as `OpenTelemetryCollector` spec.config
# (opentelemetry.io/v1beta1): native receivers / processors / exporters /
# connectors / extensions / service.pipelines, real component names, OTTL.
#
# The operator merges this fragment into the referenced central collector. On
# merge it:
#   * key-prefixes every component per tenant (filter -> filter/tenant-payments)
#     so names never collide across tenants,
#   * auto-injects a namespace gate from `source` so a tenant only ever sees
#     its own telemetry even if its pipeline forgets to filter,
#   * resolves `env` from the tenant's OWN secrets, so the platform team never
#     holds tenant backend credentials.
#
# The only fields beyond a stock collector config are: collectorRef, source, env.
# =============================================================================
apiVersion: otel.platform.io/v1alpha1
kind: TenantPipeline
metadata:
  name: payments-telemetry
  namespace: tenant-payments
  labels:
    app.kubernetes.io/managed-by: otel-multitenant-operator
spec:

  # Where this fragment is injected.
  collectorRef:
    name: gateway
    namespace: observability

  # Namespace gating. Operator generates the routing into this pipeline and an
  # enforced filter on k8s.namespace.name. Defaults to the CR's own namespace.
  source:
    namespaces:
      - tenant-payments

  # Tenant-owned secrets, surfaced to exporters as ${env:...}. Identical shape
  # to OpenTelemetryCollector spec.env (core/v1 EnvVar).
  env:
    - name: SPLUNK_HEC_TOKEN
      valueFrom:
        secretKeyRef: { name: splunk-hec-token, key: token }
    - name: NEW_RELIC_LICENSE_KEY
      valueFrom:
        secretKeyRef: { name: newrelic-license, key: license-key }

  # ---------------------------------------------------------------------------
  # NATIVE COLLECTOR CONFIG — same schema as OpenTelemetryCollector spec.config
  # ---------------------------------------------------------------------------
  config:

    receivers:
      # Common case: omit receivers entirely and inherit the gateway's, fed in
      # by the operator-generated routing connector. Shown here to mirror the
      # collector CR 1:1; the operator namespaces the bind so there's no clash.
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15

      # logs: sample to 25% (operator can be told to exempt errors via routing)
      probabilistic_sampler:
        sampling_percentage: 25

      # logs: parse structured JSON bodies into attributes
      transform/promote:
        log_statements:
          - context: log
            statements:
              - merge_maps(attributes, ParseJSON(body), "upsert") where IsMatch(body, "^\\{")

      # tenant metadata onto every record
      resource/tenant-meta:
        attributes:
          - { key: team,        value: payments, action: upsert }
          - { key: cost_center, value: "CC-4815", action: upsert }
          - { key: compliance,  value: pci,      action: upsert }

      # logs: tenant owns their own drop rules
      filter/drop-noise:
        error_mode: ignore
        logs:
          log_record:
            - 'IsMatch(body, ".*healthcheck.*")'
            - 'severity_text == "DEBUG"'

      # metrics: keep only what the tenant uses (drop everything else)
      filter/keep-metrics:
        metrics:
          metric:
            - 'not (IsMatch(name, "http.server.*") or IsMatch(name, "payments.transaction.*"))'

      # metrics: kill high-cardinality attributes before export
      attributes/drop-cardinality:
        actions:
          - { key: http.user_agent, action: delete }
          - { key: http.url,        action: delete }

      # traces: per-tenant tail sampling
      tail_sampling:
        decision_wait: 10s
        policies:
          - name: errors
            type: status_code
            status_code: { status_codes: [ERROR] }
          - name: slow
            type: latency
            latency: { threshold_ms: 500 }
          - name: baseline
            type: probabilistic
            probabilistic: { sampling_percentage: 10 }

      batch: {}

    exporters:
      splunk_hec/prod:
        token: ${env:SPLUNK_HEC_TOKEN}
        endpoint: https://splunk.payments.example.com:8088/services/collector
        source: otel
        index: payments

      awss3/archive:
        s3uploader:
          region: us-east-1
          s3_bucket: payments-telemetry-archive
          s3_prefix: logs

      otlphttp/newrelic:
        endpoint: https://otlp.nr-data.net
        headers:
          api-key: ${env:NEW_RELIC_LICENSE_KEY}

      otlp/tempo:
        endpoint: tempo.observability.svc:4317
        tls:
          insecure: true

    service:
      pipelines:
        logs:
          receivers: [otlp]
          processors:
            - memory_limiter
            - probabilistic_sampler
            - transform/promote
            - resource/tenant-meta
            - filter/drop-noise
            - batch
          exporters: [splunk_hec/prod, awss3/archive]

        metrics:
          receivers: [otlp]
          processors:
            - memory_limiter
            - filter/keep-metrics
            - attributes/drop-cardinality
            - resource/tenant-meta
            - batch
          exporters: [otlphttp/newrelic]

        traces:
          receivers: [otlp]
          processors:
            - memory_limiter
            - tail_sampling
            - resource/tenant-meta
            - batch
          exporters: [otlp/tempo]

# -----------------------------------------------------------------------------
# Operator-managed status: validation, quota, and what actually got merged.
# -----------------------------------------------------------------------------
status:
  phase: Injected
  observedGeneration: 4
  conditions:
    - { type: Validated,   status: "True", reason: ConfigSchemaValid }
    - { type: WithinQuota, status: "True", reason: SeriesUnderLimit,
        message: "est. 12k active series / 50k namespace cap" }
    - { type: Injected,    status: "True", reason: MergedIntoCollector,
        message: "rendered into observability/gateway (config hash a1b2c3)" }
  renderedPipelines:
    - logs/tenant-payments
    - metrics/tenant-payments
    - traces/tenant-payments

---
# =============================================================================
# Minimal variant — logs only, unsampled, one destination. "Only need one
# signal" is just a config with one pipeline; "no telemetry" is no CR at all.
# =============================================================================
apiVersion: otel.platform.io/v1alpha1
kind: TenantPipeline
metadata:
  name: batch-jobs-logs
  namespace: tenant-batch
spec:
  collectorRef:
    name: gateway
    namespace: observability
  source:
    namespaces: [tenant-batch]
  config:
    receivers:
      otlp:
        protocols:
          grpc: { endpoint: 0.0.0.0:4317 }
    processors:
      batch: {}
    exporters:
      loki:
        endpoint: http://loki.observability.svc:3100/otlp
    service:
      pipelines:
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [loki]
```
