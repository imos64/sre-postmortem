# Public evidence — BOOKING-2026-09-16

[Incident architecture](../README.md) · [Full report](../2026-09-16-booking-service-restarts-postmortem.md)

These are reviewed records from the September 16 investigation. The original
private collection remains separate. No Kubernetes Secrets or credential values
were requested for publication. Unrelated cluster rules, full workload manifests,
node inventories, and raw configuration snapshots are excluded.

| Evidence | What it establishes |
| --- | --- |
| [Pod](pod.txt), [deployment](deployment.txt), [final pod](final-pod.txt) | Exact UID, runtime image, previous exit 1, one replica, and stable final restart count |
| [Previous booking log](booking-previous.txt) | Pool timeout, database reachability errors, fatal `P1001` and expiry stack |
| [Current booking log](booking-current.txt) | AMQP retries, recovery and successful subsequent probes |
| [PostgreSQL log](postgres-log.txt) | Earlier database startup recovery and client connection resets; not a network diagnosis |
| [Runtime code excerpt](runtime-source-excerpt.txt) | Initial expiry query outside per-booking error handling and discarded timer promise |
| [Namespace events](events.txt) | Aggregated readiness failures across several services, plus booking starts |
| [Rule](prometheus-rules.txt) | Actual expression, five-minute hold, inactive recorded evaluation |
| [Restart counter](restart-history.json), [rolling increase](restart-increase.json) | 7 → 51 in the selected window and evaluated 15-minute increase |
| [Alert states](alert-history.json), [booking readiness](ready-history.json) | Sampled firing/pending intervals and intermittent NotReady status |
| [Postgres readiness](postgres-ready-history.json), [Postgres restarts](postgres-restart-history.json), [node readiness](node-ready-history.json) | Dependency context; not proof of uninterrupted application connectivity |
| [History summary](history-summary.json) | Derived alert intervals, 196 zero-readiness samples and monitoring gap |
| [Health](health-live.txt), [readiness](health-ready.txt), [verification metadata](verification-index.json) | Successful service-proxy GET checks at 08:24:37 UTC |
| [Workload inventory](workload-inventory.json) | Deployed application components and their captured status |
| [Ingress routes](ingress-routes.json), [gateway routes](gateway-routes.txt) | Application architecture routing, inspected later during publication preparation |
| [Capture metadata](capture-index.json), [metric queries](metrics-index.json) | Original commands, capture times, query bounds and return codes |
| [Provenance](PROVENANCE.json), [checksums](SHA256SUMS) | Per-file source/transformations and public evidence byte integrity |

The main historical snapshot was captured at 08:22:56–08:22:57 UTC, with final
verification at 08:24:37 UTC. Runtime booking code was read between these captures;
its individual command timestamp was not retained. Architecture ingress/gateway
reads happened later and carry timestamps in provenance. They add topology
context without changing historical health findings.

The original booking log requests were bounded to 2,000 previous and 3,000 current
lines; PostgreSQL to nine hours and 5,000 lines. Publication removes very long
bundled-library lines (at least 2,500 characters) and empty timestamped lines.
URL userinfo redaction from the original capture is preserved. These are selected,
transformed logs, not certified unmodified raw streams. No retained log line
records an end-user booking operation in this capture.

Prometheus queries cover 00:00–08:23 UTC with 15-second steps. These are evaluated
range-query samples, not raw scrape timestamps. The readiness estimate is
196 × 15 seconds = 49 minutes; it is not an HTTP availability SLI. There is a gap
between returned samples at 00:23:15 and 00:26:30. Alert interval edges identify
first/last returned samples, not exact notification delivery or resolution.

Capture metadata references some original outputs that were intentionally omitted
or reduced for publication; consult provenance for the published replacements.
Full `pods.txt`, `nodes.txt`, `services.txt`, `endpoints.txt`, and unrelated current
alerts are not included. Literal environment configuration is excluded. The full
report's references to such inspection describe the original read-only work.

The exact initiating connectivity cause and causes of earlier individual restarts
remain unresolved. Packet captures, Loki historical logs, database workload
inspection, notification delivery records and user request traces were not
collected. Evidence does not establish data integrity or absence of data loss.

From this directory, verify the published files with:

```bash
sha256sum -c SHA256SUMS
```

Checksums cover every evidence file except the checksum file itself. They establish
byte integrity, not independent proof of the incident.
