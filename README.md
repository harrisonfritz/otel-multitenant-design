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
- how do we ensure collector pods do not have to restart? With thousands of tenants applying changes, we need a graceful reload. How does data persist through a pipeline change?
- how do we prevent one tenant from doing an enormously expensive pipeline (I don't think fluent-bit has a way to prevent this either)

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

# Appendix / Claude design doc

# Multitenant Support for the OpenTelemetry Collector

**Status:** Draft for discussion
**Target repos:** `open-telemetry/opentelemetry-collector-contrib`, `open-telemetry/opentelemetry-operator`
**Related issues:** [collector-contrib#48895](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/48895), [opentelemetry-operator#1906](https://github.com/open-telemetry/opentelemetry-operator/issues/1906)
**Prior art referenced:** [harrisonfritz/otel-multitenant-design](https://github.com/harrisonfritz/otel-multitenant-design), [Fluent Operator](https://github.com/fluent/fluent-operator), [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)

---

## 1. Summary

Platform teams running a shared OpenTelemetry Collector ("gateway") for many application teams ("tenants") have no supported, self-service way to let those teams own their slice of the telemetry pipeline — their own sampling, filtering, attribute enrichment, and export destinations — without the platform team being in the loop for every change, and without one tenant's change being able to hurt another's telemetry.

This document deliberately does **not** lead with a single architecture. The multitenant problem is really a bundle of *separable* sub-problems, each with its own set of options and its own precedent in adjacent systems (Fluent Operator and prometheus-operator both solved versions of these, and where they got burned is instructive). Collapsing them into one "Option A vs Option B" decision hides the fact that most combinations are independently choosable. So this doc:

1. States the sub-problems separately (§4).
2. Lays out the options for each, with the Fluent and Prometheus precedent for each (§5).
3. Composes those into a small number of coherent end-to-end architectures (§6).
4. Recommends a path, and calls out where OTel *cannot* copy the precedents because its tenant workloads are fundamentally less bounded than a Prometheus scrape job (§7).

---

## 2. Motivation

The need is independently reported from two directions:

- **collector-contrib#48895** — a platform team runs one gateway Collector for a cluster with hundreds of app namespaces. Each team wants its own sampling/filtering/export config, but today that requires the platform team to hand-edit a central config per request. The `routing` connector can dispatch by attribute, but the routes are still centrally authored and redeployed on every tenant change — it relocates the bottleneck rather than removing it.
- **opentelemetry-operator#1906** — from the Operator side, `OpenTelemetryCollector` today is a single monolithic CRD owning the whole `config.yaml` as one blob. There's no supported way to compose one Collector's config from many CRs owned by many teams. The maintainer discussion explicitly reaches for the prometheus-operator model (`ServiceMonitor`-style narrow CRs aggregated into one workload object).

The common requirement: **no platform-team-in-the-loop per change, no shared restart / cross-tenant blast radius per change, and real isolation between tenants** — none of the existing primitives (routing connector, monolithic CRD, per-tenant Collector) deliver all three at scale.

---

## 3. Goals & Non-Goals

**Goals**
- Tenants declare their own pipeline via namespace-scoped CRs, gated by standard Kubernetes RBAC, with no per-change platform approval.
- The platform retains control of the boundary: which component types are allowed, resource ceilings, which destinations are reachable.
- One tenant's change never restarts or interrupts collection for another tenant.
- A malformed or malicious tenant fragment cannot crash, starve, or leak data from the shared gateway.
- Scales to O(1,000+) tenants on a single logical gateway (which may itself be sharded).
- Fits the existing Operator reconciliation model rather than requiring a parallel control plane.

**Non-Goals**
- Per-tenant Collector *processes* as the primary answer (already supported; it's the scaling ceiling this raises, and remains correct for the minority of tenants needing hard isolation).
- Backend/storage tenancy (`X-Scope-OrgID` etc. — that's Mimir/Tempo's problem). This is about pipeline *authoring*.
- Changing the `routing` connector itself, though it remains a valid low-level compile target.

---

## 4. The Problems, Stated Separately

The thing usually called "multitenancy" decomposes into nine sub-problems. They are largely orthogonal — you pick an option for each, and most combinations are valid.

| # | Problem | The core question |
|---|---|---|
| P1 | **Tenant-facing schema** | What object(s) does a tenant author? |
| P2 | **Read/merge path** | How does an authored CR become live config in the running gateway? |
| P3 | **Reconfiguration without blast radius** | How does a change take effect without restarting or interrupting other tenants? ("no restart") |
| P4 | **Resource isolation** | How is one tenant prevented from starving CPU/mem/queue from others? |
| P5 | **Config isolation** | How is one tenant prevented from reading/mutating/blocking another's data in the graph? |
| P6 | **Cross-namespace / aggregation authorization** | Which gateway picks up which tenant fragment, and who authorizes cross-namespace references? |
| P7 | **Policy / component surface** | How does the platform bound which component types and fields tenants may use? |
| P8 | **Validation & status feedback** | How does a tenant learn whether their config worked, and why not? |
| P9 | **Secret handling** | How do tenant export credentials stay owned by the tenant? |

The critical insight from the precedents (§5) is that **P8 and P4 are the ones the reference implementations either solved late or never solved — and they're the two that matter most for OTel specifically.**

---

## 5. Options Per Problem (with Fluent & Prometheus precedent)

### P1 — Tenant-facing schema

| Option | Description |
|---|---|
| 1a | **Single merged CRD** (`TenantPipeline`) mirroring the Collector config schema; one CR = whole pipeline. (harrisonfritz model) |
| 1b | **Split CRDs by concern** (`Pipeline` + `Exporter`), prometheus-operator style. |
| 1c | **Split by signal *and* concern** (traces/metrics/logs × input/filter/output) — fullest Fluent-style decomposition. |
| 1d | **Cluster-scoped + namespaced variants of each kind** (Fluent's `ClusterInput` vs `Input`), letting the platform own shared/global stages and tenants own namespaced ones. |

**Fluent:** chose 1c + 1d — separate `Input`/`Filter`/`Output`/`Parser` kinds, each with a `Cluster`-scoped variant for platform-owned stages and a namespaced variant for tenant-owned ones.
**Prometheus:** chose 1b — narrow `ServiceMonitor`, later adding `PodMonitor`, `Probe`, `ScrapeConfig`. Notably did **not** do 1d (no `ClusterServiceMonitor`); it controls scope from the selecting side instead (see P6). The set of kinds *grew over time*, which is direct evidence that composability must be designed in — you will need more kinds than you first ship.

### P2 — Read/merge path

| Option | Description |
|---|---|
| 2a | **Operator reconciles CRs → ConfigMap/Secret → reload.** |
| 2b | **In-process custom `confmap` provider** — the Collector reads CRs itself via informer and reloads; no operator merge hop. |
| 2c | **Hybrid** — operator does validation + status writeback; provider does the live merge. |

**Fluent:** 2a — the operator watches CRs, regenerates the fluent-bit config, writes it to a Secret/ConfigMap, and a reloader triggers reload.
**Prometheus:** 2a, strictly — the generated config lives in a Secret named after the Prometheus object, mounted into the pod; a config-reloader sidecar hits Prometheus's reload endpoint. Prometheus **never reads CRs itself**; the operator is always the intermediary. This is a deliberate, at-scale-proven choice to keep the *workload* process CR-unaware and confine all Kubernetes API contact (and RBAC) to the operator — a direct argument against 2b on blast-radius and permission grounds.

> **On Option 2b (custom `confmap` provider):** technically viable — the Collector's `confmap.Provider` interface already supports informer-driven `watcher.OnChange()` reloads, and multiple `--config` sources already merge natively, so a `tenantconfig:` provider could compose platform base config with dynamic tenant fragments. But it (a) doesn't remove controller-side code, because status feedback (P8) still needs a writeback path the in-process provider doesn't have; (b) forces a custom Collector build via OCB, sharpening version-coupling (P7/§8); and (c) gives the Collector process direct K8s list/watch RBAC that neither Fluent nor Prometheus grant their workloads. Recommendation: **do not lead with 2b.** It's an optimization to keep in reserve for propagation latency, not a foundation.

### P3 — Reconfiguration without blast radius ("no restart")

| Option | Description |
|---|---|
| 3a | **Full process restart** (Operator default today) — unacceptable for a shared gateway. |
| 3b | **Whole-config in-process hot-reload** — reload all pipelines, no pod eviction, brief internal churn. |
| 3c | **Component-granular hot-reload** — swap only the changed tenant's components. Does not exist in Collector core today; largest lift. |
| 3d | **Blue/green or canary replicas** — roll a replacement, drain the old. |
| 3e | **Shard by tenant hash** — bound any reload/restart to one shard. |

**Fluent:** 3b — fluent-bit hot-reload re-reads the whole config; no per-tenant granularity.
**Prometheus:** 3b — the reloader triggers a whole-config reload via the reload endpoint; no component-granular reload, no operator-built sharding. **This is exactly where it bites:** an invalid `ServiceMonitor` could pass CRD schema validation, the operator would detect the change, and Prometheus would then fail to reload the *entire* config — and if this happened at startup, Prometheus wouldn't start at all. One tenant's bad fragment, whole-gateway blast radius. Prometheus survived this only by (a) hardening validation heavily over many releases (P7/P8) and (b) the fact that a scrape target is cheap. **Neither mitigation is free for OTel** — see §7.

> Pragmatic v1 for OTel: **3b + 3e** — whole-config reload, but "whole" is one shard's worth of tenants, so blast radius is bounded to shardmates. 3c is the clean long-term answer and is real Collector-core work worth tracking separately.

### P4 — Resource isolation

| Option | Description |
|---|---|
| 4a | **Static admission-time ceilings** — reject fragments exceeding declared limits. |
| 4b | **Runtime per-tenant metrics + throttle loop** — tenant-labeled `otelcol_*` feeding automatic throttling. |
| 4c | **Shard by tenant hash** — noisy tenant only starves its shardmates (same mechanism as 3e). |
| 4d | **Per-tenant `memory_limiter` + queue-per-exporter** — structural, limited by what the pipeline model exposes today. |
| 4e | **Kubernetes-level isolation** — dedicated node pools / quotas per shard, punting to the scheduler. |

**Fluent:** effectively 4e + "fragments are cheap" — log routing/filtering is relatively bounded, so this was never a first-order problem.
**Prometheus:** essentially **unsolved within one process**, and it mostly didn't have to be — a scrape target is cheap and bounded, so a noisy `ServiceMonitor` is one more scrape job, not an unbounded graph. High object counts have caused CPU/memory pressure, but there is no per-tenant resource accounting inside Prometheus; the scaling answer was operational (run more instances split by selector = 4c/4e).

> **This is the one place OTel is fundamentally harder than either precedent** (see §7). Neither Fluent nor Prometheus gives you a copyable answer, because their tenant fragments are cheap; a Collector tenant can attach tail-sampling or high-cardinality transforms with wildly unequal, unpredictable cost. 4a-alone is insufficient; 4c is the realistic primary mitigation, with 4a as a guardrail and 4b as an observability layer.

### P5 — Config isolation

| Option | Description |
|---|---|
| 5a | **Name-prefix / namespace all component IDs** (`{ns}_{name}_{type}`) — tenant never sees or can address another's components. Structural. |
| 5b | **Separate pipeline instances per tenant within one process** — stronger, if the runtime supported it (related to 3c). |

**Fluent & Prometheus:** both rely on 5a — the operator prefixes/encodes the CR's identity into the generated config so names can't collide across tenants. This is the baseline any merge-based approach must do; it's well-proven and low-risk.

### P6 — Cross-namespace / aggregation authorization

| Option | Description |
|---|---|
| 6a | **`ReferenceGrant`-style object** (Gateway API convention) — target namespace grants access. |
| 6b | **Inline `allowedConsumers` selector** on the target Exporter. |
| 6c | **Disallow cross-namespace in v1** — same-namespace refs only; simplest. |
| 6d | **Platform-side label/namespace selector** — the gateway object decides what it pulls in; the tenant neither declares a target nor grants anything. |

**Fluent:** 6d — the `FluentBitConfig`/selector on the platform side decides which namespaced CRs are aggregated.
**Prometheus:** 6d — `ServiceMonitor`s and their namespaces are selected by the `serviceMonitorSelector` + `serviceMonitorNamespaceSelector` on the platform-owned `Prometheus` object. The tenant grants nothing; the platform selects. Clean and proven, and maps directly onto the `pipelineSelector` sketched in §6/§9. **Baked-in tradeoff:** a tenant can apply a perfectly valid CR that is silently ignored because no gateway selector matches it — a well-documented, hours-lost-to-it footgun. Design an explicit "unclaimed / unselected" status for this, which Prometheus historically lacked.

### P7 — Policy / component surface

| Option | Description |
|---|---|
| 7a | **Inline `policy` fields** on the platform/gateway CR (allowed types, field restrictions, ceilings). |
| 7b | **External OPA/Kyverno policy** objects — reuse cluster policy tooling. |
| 7c | **RBAC-per-CRD-kind** — grant `Pipeline` but withhold `Exporter`, etc. (only available if P1 = split CRDs). |
| 7d | **Allow-list baked into the operator/provider build** — compile-time restriction of registered components; the binary can't load a disallowed component. |

**Fluent:** primarily 7c + 7d — cluster-vs-namespaced kinds gate authorship, and the built image determines available plugins.
**Prometheus:** historically **weak, and a cautionary tale.** CRD validation was described as very new / not yet implemented, so invalid `ServiceMonitor`s could be created and would then break Prometheus's reload. Hardening came incrementally over many releases (miscellaneous `ScrapeConfig` validations, version-specific rejections). **Lesson:** bake the allow-list and field-level restriction (7a/7d) in from `v1alpha1`; retrofitting validation onto CRs that already accept anything is a multi-year cleanup.

### P8 — Validation & status feedback

| Option | Description |
|---|---|
| 8a | **Async controller writes `status.conditions`** back onto the tenant's CR. |
| 8b | **Synchronous validating admission webhook** — tenant rejected at `kubectl apply` with an immediate reason. |
| 8c | **Both** — webhook for structural/policy failures, status for semantic/runtime failures. |

**Fluent:** reconcile-time validation, operator logs, limited status.
**Prometheus:** the most revealing comparison. For most of its life, prometheus-operator had **no status writeback to `ServiceMonitor`** — the feedback loop was "grep the generated Secret to see if your monitor was picked up," read operator logs, or watch the `PrometheusOperatorRejectedResources` alert. A status subresource for `ServiceMonitor`/`PodMonitor`/`Probe`/`ScrapeConfig` arrived only recently, behind the `StatusForConfigurationResources` feature gate, and only for Prometheus workloads. The admission webhook exists but historically only for `PrometheusRule`, not the monitor CRDs.

> **This is the strongest single lesson for OTel.** The reference implementation ran for *years* with exactly the "apply and get silence" gap — and the ecosystem's verdict (the recent status work; the sheer volume of troubleshooting guides that exist *because* of this gap) is that it was a real, long-standing problem they eventually had to close. Build **8c from the start.** Self-service isn't "can I submit config" — it's "can I tell whether it worked and why not." Without P8, everything else is a trap.

### P9 — Secret handling

| Option | Description |
|---|---|
| 9a | **`secretRef` resolved in the tenant's own namespace** — tenant keeps ownership. |
| 9b | **Platform-managed shared destinations** — tenant references a named platform Exporter, never handles credentials (pairs with 7c to withhold `Exporter` creation). |
| 9c | **Projected/mounted per-exporter volumes** vs. controller-read-then-inject — a plumbing sub-choice under 9a. |

**Fluent & Prometheus:** both do 9a — secret references resolve in the CR's namespace; the operator/workload reads only the referenced Secret. `OpenTelemetryExporter` is the natural secret-ownership boundary for OTel.

---

## 6. Composing the Options into Architectures

Three coherent end-to-end columns fall out. Each is a full pick across P1–P9.

| | **Arch A — Single CRD** | **Arch B — Split CRDs (Prometheus-shaped)** | **Arch C — Provider-driven** |
|---|---|---|---|
| P1 schema | 1a `TenantPipeline` | 1b/1c `Pipeline`+`Exporter`(+`Gateway`) | either 1a or 1b |
| P2 merge | 2a operator→ConfigMap | 2a operator→ConfigMap | 2b/2c in-process provider |
| P3 reload | 3b+3e | 3b+3e | 3b+3e |
| P4 resources | 4a+4c | 4a+4c | 4a+4c |
| P5 config iso | 5a | 5a | 5a |
| P6 aggregation | 6c→6d | 6d (`pipelineSelector`) | 6d |
| P7 policy | 7a (coarse) | 7a+7c+7d (fine) | 7a+7d |
| P8 feedback | 8c | 8c | 8c (still needs external webhook + status-writer) |
| P9 secrets | 9a | 9a/9b | 9a |
| **Upstream fit** | bespoke, harder to land | **matches #1906 direction** | novel; needs custom build |
| **Impl cost** | lowest | medium | medium-high |
| **Debuggability** | one object/tenant | needs cross-CRD status rollups | worst (feedback split across artifacts) |

Notes:
- **Arch A** is the fastest first step and the smallest API surface, but mirroring the whole Collector schema in one CRD is an ongoing lockstep-maintenance burden, isolation is convention-enforced rather than structural, and it doesn't map onto the Operator's CRD taxonomy — harder to land *upstream*.
- **Arch B** matches where the Operator maintainers were already heading in #1906 and where prometheus-operator ended up; P7/P8 are structurally easier with narrow, independently-versioned CRDs. Higher up-front cost; a single logical pipeline is no longer visible in one `kubectl describe`, so cross-CRD status tooling is required from day one.
- **Arch C** removes an operator hop (faster propagation) but does **not** remove controller-side code — P8 still forces an external webhook + status-writer — and adds a custom-build/version-coupling burden neither precedent took on. Keep in reserve.

---

## 7. Recommendation

**Adopt Arch B (split CRDs, Prometheus-shaped), and treat P4 and P8 as first-class rather than deferred.**

Rationale:
- It's the direction Operator maintainers independently reached in #1906, and prometheus-operator proves the spine — split CRDs + platform-side selector aggregation + operator-mediated reload — works at very large scale.
- The two requirements that dominate once real tenants are on it — bounded component surface (P7) and version safety (P8) — are structurally easier with narrow, independently-versioned CRDs than with one CRD mirroring the whole schema.

**But the precedents come with two caveats that matter *more* for OTel than they did for their originators, and these are the persuasive core of this proposal:**

1. **Whole-config reload made one bad fragment everyone's problem (P3/P8).** Prometheus survived this only because it hardened validation heavily *and* because scrape targets are cheap. OTel gets neither for free: tenant fragments are arbitrarily expensive, so you must ship strong admission-time validation (8b/8c) **and** sharding (3e) from v1 — not as follow-ups. Copying Prometheus's spine without copying the decade of validation-hardening that made it safe would reproduce its worst failure mode on day one.

2. **The status/feedback gap was real and took Prometheus years to close (P8).** Do not repeat that detour. 8c (webhook + status subresource) is a v1 requirement, because self-service is defined by legibility of outcome, not by the ability to submit.

**The one place no precedent helps: resource isolation (P4).** A Prometheus `ServiceMonitor` or a Fluent `Filter` is cheap and bounded; an OTel `Pipeline` with tail-sampling or high-cardinality transforms is not. This is the sub-problem that needs original design — recommended: 4c (shard by tenant hash) as the primary blast-radius bound, 4a (static ceilings at admission) as a guardrail, 4b (per-tenant metrics) as the observability layer — and it's the part most worth a dedicated spike before this goes upstream.

**On sequencing:** it is defensible to ship an Arch-A-shaped `v1alpha1` first for speed and split into Arch B later — but only if the split can be done without breaking the tenant-facing API. Prometheus's own history (kinds added incrementally, validation retrofitted painfully) argues for designing the split up front instead. Open question for maintainers: design the split now, or ship single-CRD alpha and split based on observed pain?

---

## 8. Cross-cutting Risk: Version Coupling (P7/P8)

Called out explicitly in #1906: a tenant fragment authored against one Collector/component version can silently break when the platform upgrades the gateway image. Prometheus hit the same class of problem (version-specific config rejections appearing in its changelog over time). Mitigations, in preference order: (a) independently-versioned CRDs with explicit compatibility ranges (Arch B enables this); (b) admission-time validation that knows the target gateway's component versions; (c) status conditions surfacing "incompatible with current gateway version" back to the tenant. Arch C worsens this by adding a custom-build compatibility axis on top.

---

## 9. Illustrative CRD Sketch (Arch B)

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryExporter          # tenant-owned; natural secret boundary (P9)
metadata: { name: team-a-datadog, namespace: team-a }
spec:
  type: otlphttp
  config:
    endpoint: https://otlp.datadoghq.com
    headers:
      DD-API-KEY: { secretRef: { name: team-a-dd-key, key: api-key } }
  allowedConsumers:                  # P6: opt-in for cross-namespace refs (6b); omit for same-ns only (6c)
    namespaceSelector: { matchLabels: { team: a } }
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryPipeline          # tenant-owned
metadata:
  name: team-a-traces
  namespace: team-a
  labels: { gateway: cluster-gateway }   # P6: how the platform selector claims it
spec:
  signal: traces
  receivers:  [ { type: otlp, config: {} } ]
  processors:
    - { type: batch }
    - type: attributes
      config: { actions: [ { key: tenant, value: team-a, action: insert } ] }
  exporterRefs: [ { name: team-a-datadog } ]   # same-ns default
  status:                            # P8: written back by controller
    conditions: [ { type: Accepted, status: "True" } ]
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryGateway           # platform-owned
metadata: { name: cluster-gateway, namespace: observability }
spec:
  pipelineSelector: { matchLabels: { gateway: cluster-gateway } }   # P6: 6d, Prometheus-style
  sharding: { strategy: tenant-hash, replicas: 8 }                  # P3/P4: 3e + 4c
  policy:                            # P7: allow-list (7a), enforced at admission (8b)
    allowedReceiverTypes:  [ otlp ]
    allowedProcessorTypes: [ batch, attributes, filter, transform ]
    maxQueueSizePerTenant: 5000
    maxCPUMillicoresPerTenant: 250
```

Reconciliation: the `OpenTelemetryGateway` controller lists `Pipeline`s matching `pipelineSelector`, resolves `exporterRefs` (same-ns default; cross-ns only if the target's `allowedConsumers` matches), validates each against `policy` (rejecting with a status condition on the *tenant's own* object, never blocking others' reconciliation), name-prefixes all component IDs (5a), assigns each tenant to a shard by hash (3e/4c), generates per-shard config, and applies via the existing Operator config path with whole-config reload (3b). A validating webhook enforces the same `policy` synchronously at apply time (8b); the controller writes async semantic/runtime status (8a) — together, 8c.

---

## 10. Open Questions

1. **Cross-namespace auth (P6):** `ReferenceGrant`-style object (Gateway API consistency) or inline `allowedConsumers` selector (simpler first)? Or disallow entirely in v1?
2. **Where does P4 live** — Collector core (new per-pipeline resource accounting), the Operator (sharding only), or an admission-time cost-estimation webhook? Determines how much needs Collector-core buy-in vs. Operator-only.
3. **Sequencing:** design the Arch-A→Arch-B split up front, or ship single-CRD alpha and split on observed pain? Is a lossless one-way `TenantPipeline`→`Pipeline`+`Exporter` conversion tool sufficient?
4. **Multi-gateway fan-out:** may the same `Pipeline` be selected by more than one `Gateway`? If so, which gateway's validation status is authoritative?
5. **Policy mechanism (P7):** inline `Gateway.spec.policy`, or external OPA/Kyverno objects to reuse existing cluster policy tooling?
6. **New Kind vs. extension:** is `OpenTelemetryGateway` a new Kind, or a `pipelineSelector`/`sharding` extension to the existing `OpenTelemetryCollector` CRD?
7. **Onboarding/discovery:** how does a new tenant learn which component types and destinations are even available to them (surfacing the P7 allow-list)?

---

## 11. Rollout Plan (proposed)

1. Ship `OpenTelemetryPipeline` + `OpenTelemetryExporter` as `v1alpha1`, same-namespace only (defers Q1), with the allow-list `policy` baked in from the start (P7) and both webhook + status (P8/8c) — **not** deferred, per §7.
2. Ship `OpenTelemetryGateway` (or `OpenTelemetryCollector` extension — Q6) with `pipelineSelector` (6d) and static ceilings (4a).
3. Land sharding-by-tenant-hash (3e/4c) as the initial blast-radius bound — the mitigation Prometheus never built and OTel can't skip.
4. Graduate to `v1beta1` after ≥1 real multi-hundred-tenant deployment; use that to settle Q2–Q4.
5. Track Collector-core component-granular hot-reload (3c) as longer-term follow-on that would let a future `v1` relax the sharding requirement.

---

## 12. References

- [collector-contrib#48895 — Support Multitenant Kubernetes Self-Service](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/48895)
- [opentelemetry-operator#1906 — Distributed collector configuration](https://github.com/open-telemetry/opentelemetry-operator/issues/1906)
- [harrisonfritz/otel-multitenant-design](https://github.com/harrisonfritz/otel-multitenant-design)
- [Fluent Operator](https://github.com/fluent/fluent-operator) — CRD-per-concern, cluster+namespaced variants, operator→config→reload
- [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) — `ServiceMonitor`-style narrow CRDs, platform-side selector aggregation, Secret+reloader, late-arriving status subresource

