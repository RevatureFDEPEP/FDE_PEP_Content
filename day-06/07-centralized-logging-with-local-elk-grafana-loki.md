# Centralized Logging with Local ELK / Grafana Loki

> *Day 6 — Week 2, Monday*

## Overview

This file is orientation, not deployment. You will not stand up ELK or Loki on this course. But you will, very soon, hit the wall that every microservice developer hits: `docker compose logs` is fine when you have one container, manageable when you have four, and useless when you have ten. The reverse proxy you built today made it worse — every request now traverses multiple services, and tracking one request through all of them by tailing five separate log streams in five terminal panes is a losing game.

The purpose of this file is to leave you with a clear mental model of what a centralized logging stack looks like, *why* it exists, and where it would slot into the substrate you are building. When you see Datadog, Splunk, ELK, Loki, or OpenSearch on a job in the future, you should recognize the shape.

## The three-part shape of every logging stack

Every centralized logging system, regardless of vendor, has three components:

1. **Collector / shipper.** Runs near each application (sidecar, daemonset, or agent on the host). Reads logs from stdout/stderr or from files. Adds metadata (container name, image, environment). Ships to the store.
2. **Store.** Receives shipped logs, indexes them, persists them. Provides a query API. This is where the heavy lifting (and the operational cost) lives.
3. **Viewer / UI.** Talks to the store's query API. Lets humans search, filter, build dashboards, set up alerts.

Two well-known instantiations of this shape:

| Shape          | Collector             | Store          | Viewer        | Notes |
|----------------|-----------------------|----------------|---------------|-------|
| **ELK / Elastic** | Filebeat / Logstash | Elasticsearch  | Kibana        | The classic. Powerful full-text search, expensive at scale (indexes everything). |
| **Grafana Loki** | Promtail / Fluent Bit | Loki           | Grafana       | "Logs without indexing the content." Indexes labels only — cheaper, queries are line-grep over time windows. |

There are others — OpenSearch (Elasticsearch fork), Splunk (closed-source, enterprise heavyweight), Datadog/New Relic (SaaS), Vector (collector that can ship to anything) — but they all map to the same three-part shape.

## Why it matters for microservices

With one monolith, `tail -f /var/log/app.log` works. With microservices, you lose three things:

1. **Locality.** Logs are spread across many containers, possibly on many hosts. You cannot tail them all.
2. **Correlation.** A single user action produces log lines in user-service, question-management, the api-gateway, and the proxy. Linking them requires a shared identifier propagated through every call.
3. **Persistence.** Container logs are ephemeral — when a container exits, its logs vanish (unless captured). Restart the stack and yesterday's logs are gone.

Centralized logging solves all three. Logs from every container land in one store with a shared timestamp axis; a request ID (injected by the proxy, propagated through every backend call) ties related lines together; the store outlives the containers.

## Request ID propagation — the half you control

Centralized logging is necessary but not sufficient. Even with all your logs in one place, you need a way to filter to "show me everything for *this* request." That requires a request ID:

1. **Proxy injects it.** On every incoming request, the proxy generates a UUID and adds it as a header:
   ```nginx
   # Inside the server block
   proxy_set_header X-Request-ID $request_id;
   add_header X-Request-ID $request_id;   # echo back to client too
   ```
   Nginx 1.11+ has a built-in `$request_id` variable that produces a unique 32-char hex string per request. Free, deterministic-per-request, no extra middleware.

2. **Backends propagate it.** Each service reads `X-Request-ID` from the incoming request, includes it in every log line, and forwards it on any outbound call to another service.

3. **Backends log it as structured field.** Not buried in a message string — as a top-level JSON field. Then the logging store can filter on it.

Wire this up *before* you wire up the logging stack. The request ID is the half you cannot fix retroactively.

## Where a logging stack would slot into the PEP substrate

If you were to deploy one (which, again, is not required), the topology would look like:

```
      ┌──────────┐
      │  Browser │
      └────┬─────┘
           │ HTTPS
      ┌────▼──────┐
      │   proxy   │────────► stdout logs ──┐
      └─┬─────────┘                        │
        │                                  │
   ┌────▼─────┐ ┌─────────┐ ┌────────────┐ │
   │  user-   │ │question-│ │   test-    │ │
   │ service  │ │  mgmt   │ │   mgmt     │ │
   └─┬────────┘ └─┬───────┘ └─┬──────────┘ │
     │            │           │            │
     └──── stdout logs ──────►├────────────┤
                              │            │
                       ┌──────▼──┐    ┌────▼─────┐
                       │ Promtail│    │ Filebeat │
                       │   or    │    │    or    │
                       │Fluent Bit    │ Logstash │
                       └──────┬──┘    └────┬─────┘
                              │            │
                       ┌──────▼──────┐ ┌───▼───────────┐
                       │    Loki     │ │ Elasticsearch │
                       └──────┬──────┘ └───┬───────────┘
                              │            │
                       ┌──────▼──┐    ┌────▼─────┐
                       │ Grafana │    │  Kibana  │
                       └─────────┘    └──────────┘
```

The key insight: the collector is the **only** new component that touches each application service. Backends just log to stdout (as good twelve-factor apps already do). The collector picks logs up via the Docker logging driver or a tail of the container's log file, ships them, and the backend never knows.

You add or remove collector/store/viewer without touching the application services at all. That clean separation is what makes the pattern adoptable.

## A minimal Loki Compose snippet (for orientation only)

Not for the deliverable — for shape recognition:

```yaml
services:
  loki:
    image: grafana/loki:3.1.0
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:3.1.0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./infra/promtail.yaml:/etc/promtail/config.yaml:ro
    command: -config.file=/etc/promtail/config.yaml

  grafana:
    image: grafana/grafana:11.1.0
    ports:
      - "3000:3000"
```

Three containers, one config file, and you have a working dev logging stack. Loki indexes by Docker labels (service name, container name, image) and stores log lines as opaque blobs — cheaper than ELK but with weaker full-text search. For development scale, it is more than adequate. The ELK equivalent has roughly the same number of containers but ~5x the memory footprint and more knobs to tune.

## What to take from this file

You should leave today able to:

- Sketch the three-part collector / store / viewer shape on a whiteboard.
- Explain why microservice architectures need centralized logging.
- Recognize ELK and Loki as instantiations of the same pattern, with different cost/feature trade-offs.
- Identify where the logging stack would attach to the PEP substrate.
- Wire up `X-Request-ID` propagation in the proxy now, even before any logging stack exists, because the propagation is the harder half.

## Common Pitfalls

- **Deploying a logging stack before you propagate request IDs.** You will have everything-in-one-place logs that you still cannot correlate. Do the request-ID work first.
- **Logging unstructured text.** `console.log("user " + userId + " did " + action)` is unsearchable. Log structured JSON with named fields and let the store index them.
- **Indexing PII or secrets.** Once it is in the store, retention policies apply. Do not log raw passwords, tokens, full credit card numbers — even on "dev" stacks, because dev stacks have a way of getting copied to production.
- **Confusing logs with metrics with traces.** Logs are discrete events with text; metrics are numeric time-series; traces are causal chains of spans. They overlap, but a logging stack does not replace Prometheus or OpenTelemetry. Know which question you are asking.
- **Assuming "centralized logging" means "one tool."** Most real shops run two or three: a logs system (Loki/ELK), a metrics system (Prometheus/Datadog), and a tracing system (Jaeger/Tempo). They are complementary.

## Key Takeaways

- Every centralized logging system is a collector + store + viewer. Vendor differences are details.
- The pattern exists because microservice topologies make per-container log tailing untenable.
- Request ID propagation is half the problem; the logging stack is the other half. Do the propagation now.
- Structured JSON logs to stdout, picked up by a Docker-aware collector, is the modern shape. Backends never need to know which logging stack is downstream.

---

*Prerequisites: Day 2 (twelve-factor stdout logging), Day 6 — Path-based routing (the proxy is where you generate request IDs).*
