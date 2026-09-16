**SRE postmortem: recurring booking-service restarts in kind-dev**

[Application architecture and incident guide](README.md) · [All incidents](../../README.md)

**Incident reference:** BOOKING-2026-09-16. **Report date:** 2026-09-16. **Status at the recorded verification:** recovered; corrective work and underlying dependency investigation remain open. **Alert severity:** warning. **Environment:** development. **Accountable team:** platform, as identified by the alert; application and database owners below are proposed assignments, not accepted commitments.

**Executive assessment.** The exact pod identified in the alert experienced recurring restarts and intermittent readiness loss. Prometheus records its main-container restart counter increasing from 7 to 51 between 00:00 and 08:23 UTC on September 16: 44 additional restarts, not 51 restarts within fifteen minutes. Six firing intervals are visible in the sampled alert history. The latest retained failure is attributable with high confidence to an unhandled database-query rejection in `expireUnpaidBookings()`: PostgreSQL became unreachable from the application, Prisma raised `P1001`, and Node.js exited with code 1. Inspection of the JavaScript in the running container confirms that the initial query is outside the per-booking error handler and the timer discards the returned promise.

Kubernetes restarted the process automatically. It became Ready at 08:01:25 UTC and remained at 51 restarts through the final check at 08:24:37 UTC. The alert rule was inactive at its 08:22:46 evaluation. This establishes current recovery, not a durable fix. No application, infrastructure, database, or deployment changes were performed during this investigation. This publication preserves the September 16 observations; it is not a later live-health assessment.

The trigger for the database reachability failure is **not yet established**. PostgreSQL did not restart during the latest failure, several other applications recorded contemporaneous readiness timeouts, and booking-service encountered RabbitMQ connection failures during recovery. These support investigating a shared dependency or connectivity problem; they do not prove a CNI, DNS, host, database-capacity, or packet-loss root cause. Earlier restart causes are not individually established by the one previous-container log retained here.

**Identity and scope.** The live pod UID matches the supplied alert exactly.

| Field | Value |
|---|---|
| Cluster / namespace | `kind-dev` / `fixam4me` |
| Pod | `booking-service-9fbfb8986-zbqdf` |
| UID | `a933dc48-d26a-4c98-abc2-b817182cddfc` |
| Container / node | `booking-service` / `dev-worker` |
| Deployment / desired replicas | `booking-service` / 1 |
| Image tag | `ghcr.io/fixam4me/booking-service:sha-fb6de2bb9a4f2cd0ae057e626c92d01e30141522` |
| Runtime image digest | `sha256:497896ac9460368552063ccb216da622a2eab21baafa73957284e23a3bef1261` |
| Runtime versions from crash log | Node.js 22.23.2; Prisma Client 5.22.0 |
| Pod creation | 2026-09-03 23:47:16 UTC |
| Metrics source | `kube-state-metrics.monitoring.svc.cluster.local:8080`, job `kube-state-metrics` |

The image tag resembles a source revision, but no source-repository-to-image attestation was verified. Causal code evidence comes from the running image itself. The init container's separate lifetime restart count of 24 is not included in the booking-service counter or the 44-restart calculation.

**Impact.** The deployment has one replica. During its observed NotReady periods it had no second booking-service replica to carry traffic. Requests requiring booking-service could therefore fail or time out, and the in-process booking expiry task and message consumer could be interrupted. Actual failed request counts, affected users, delayed bookings, payment outcomes, and financial impact were not measured. No production impact is established by this development incident.

A 15-second range query returned 196 NotReady samples among 2,001 available readiness samples. Multiplying those samples by the query step gives approximately **49 minutes of sampled NotReady time** across the 8-hour-23-minute query window. This is a diagnostic approximation, not a measured HTTP availability SLI or exact outage duration: scrape timing, query lookback, missed short transitions, and a gap between 00:23:15 and 00:26:30 limit precision. A single continuous outage must not be inferred. Nine distinct pods, including booking-service, have retained readiness-timeout events near 07:58–08:00 UTC; this is evidence of correlated symptoms, not proof that all nine services were unavailable for the same duration.

There is no verified evidence of lost or corrupted booking data, but database integrity, booking/payment reconciliation, and queue redelivery were not tested. Consequently, “no data loss” is not an accepted finding.

**Timeline.** All times below are UTC on 2026-09-16 unless a date is stated. New York was UTC−04:00; for example, the latest exit at 08:00:53 UTC was 04:00:53 EDT. Metric timestamps identify sampled observations and can lag actual transitions.

| Time | Evidence-backed observation |
|---|---|
| 00:00:00 | First sample in the selected window: booking restart counter 7. This is the query boundary, not an established incident start. |
| 00:23:15–00:26:30 | Gap in returned pod/node metric samples; host or cluster outage cause not established. |
| 00:25:53–00:25:59 | PostgreSQL starts, reports an interrupted prior shutdown, performs automatic recovery, and accepts connections. This precedes the latest failure by hours. |
| 00:26:45 | Booking restart counter first sampled at 8; rises to 9 at 00:27:45. |
| 03:17:45 | First sampled firing state in the selected window. Earlier pending states exist. |
| 07:46:46 | The previous booking container starts; counter subsequently sampled at 50. |
| 07:46:47 | Previous process connects to AMQP, subscribes, and begins listening. |
| 07:58:08.744 | Readiness logs report a Prisma pool acquisition timeout: 10-second timeout, connection limit 21. This does not establish server-wide connection exhaustion. |
| 07:58:13.784 | Readiness logs report inability to reach `postgres:5432`. |
| 08:00:23–08:00:44 | Additional database reachability errors in readiness logs. |
| 08:00:53.749 | `prisma.booking.findMany()` in `expireUnpaidBookings` raises `P1001`; last container state records exit 1, reason `Error`, at 08:00:53. |
| 08:00:54 | Kubernetes starts the replacement container in the same pod. |
| 08:01:00–08:01:13 | Three AMQP connection attempts fail and are retried. |
| 08:01:17.677 | AMQP connects on attempt 4; application startup proceeds. |
| 08:01:25 | Kubernetes Ready/ContainersReady conditions transition to True. The exit-to-Ready interval is 32 seconds; this excludes degradation before the exit. |
| 08:22:46 | Restart alert rule evaluates inactive, with rule health `ok`. |
| 08:22:56–08:22:57 | Main evidence capture: one available replica, 51 restarts, previous exit 1, current process running. |
| 08:24:37 | Final snapshot still records 51 restarts; service-proxy health and readiness requests succeed. |

Sampled firing periods were 03:17:45–03:27:15, 05:28:15–05:37:45, 06:45:00–06:53:15, 07:18:00 alone, 07:37:45–07:40:15, and 07:48:30–07:58:00. These are first/last returned firing samples, not exact notification delivery or resolution times. The supplied notification lacks a timestamp, so it cannot be assigned conclusively to one interval. A restart at 08:00:53 after the last sampled firing period demonstrates that alert clearance is not equivalent to elimination of the failure.

**Detection and response.** The installed rule is:

```promql
increase(kube_pod_container_status_restarts_total[15m]) > 3
```

It has a five-minute `for` duration and no `keep_firing_for` extension. The alert must satisfy the expression continuously for five minutes before firing. Prometheus `increase` extrapolates counter changes; the expression should not be interpreted as an exact integer event count at every evaluation. The maximum evaluated increase in the queried window was approximately 5.08. The configured labels are warning/platform, consistent with the supplied notification.

The retained alert history confirms detection of restart bursts. Notification delivery latency, acknowledgement time, incident declaration time, and MTTA are unavailable. Exact MTTD and MTTR cannot be calculated without an established start and durable resolution. The 32-second latest exit-to-Ready interval is an observed recovery interval only. The five-minute hold and fifteen-minute restart window also mean this alert cannot serve as a complete availability monitor for a singleton service: isolated restarts and readiness-only failures can occur without firing.

**Causal analysis.** The latest crash is supported by three independent artifacts: the pod's termination state, its previous process log, and the deployed JavaScript excerpt.

```text
PrismaClientKnownRequestError:
Invalid `prisma.booking.findMany()` invocation:
Can't reach database server at `postgres:5432`
at async expireUnpaidBookings (file:///app/dist/index.js:433:19)
code: 'P1001'
```

The deployed scheduler invokes:

```javascript
setInterval(() => { void expireUnpaidBookings(); }, 60_000);
```

The function awaits `prisma.booking.findMany(...)` before entering a loop whose per-booking work has a `try/catch`. Therefore that catch cannot handle failure of the initial query. `void` discards the promise without attaching a rejection handler. The combination of this code, the fatal stack, and exit 1 establishes an unhandled rejection as the high-confidence mechanism of the latest process failure. Node documents `throw` as the default unhandled-rejection mode; Prisma defines `P1001` as a database reachability error. [Node.js documentation](https://nodejs.org/download/release/v22.18.0/docs/api/cli.html#--unhandled-rejectionsmode), [Prisma error reference](https://docs.prisma.io/docs/orm/reference/error-reference).

The resulting chain is: database query fails → scheduler promise rejects without handling → application process exits → singleton service loses its process → Kubernetes restarts it → AMQP retries delay startup → readiness returns after dependencies respond. The durable application defect is the missing error boundary around the complete recurring task. The initiating database connectivity failure remains an open causal branch.

| Candidate cause or contributor | Assessment |
|---|---|
| Missing recurring-task rejection handler | Confirmed in deployed code; high-confidence cause of latest exit. |
| Database connectivity failure | Confirmed from this application's perspective; precise infrastructure cause unproven. |
| Database pool contention | Pool timeout observed; exhaustion, leak, blocked queries, and connectivity starvation not distinguished. |
| Shared connectivity/dependency disruption | Plausible given other readiness failures and AMQP retries; requires correlated network and dependency evidence. |
| PostgreSQL restart at latest exit | Not supported: its counter stays at 9 after approximately 00:26:30. Process readiness alone does not establish reachability from booking-service. |
| OOM kill | Not the recorded latest termination: `Error`, exit 1, rather than `OOMKilled`. Earlier exits not exhaustively classified. |
| Liveness-probe kill | Earlier liveness connection-refused events exist, but the latest failure is accompanied by a fatal application stack; no retained Killing event establishes a probe-triggered latest exit. |
| Bad rollout immediately before latest failure | No evidence: the pod/template dates to September 3; no rollout mitigation occurred in this investigation. |

Contributors include a single replica, a recurring task sharing the HTTP/message-consumer process, and incomplete service naming (`service: "unknown"`) in the logs. The readiness probe uses a one-second timeout while the observed Prisma pool timeout is ten seconds. That mismatch can produce probe timeouts before useful application diagnostics complete; it is not established as the cause of the application crash. The one-minute interval also lacks a visible overlapping-run guard, which is a design risk rather than an observed explanation for this incident.

**Recovery and current verification.** Recovery was automatic through Kubernetes restart and application AMQP retry behavior. No manual repair is claimed.

| Check | Result and limit |
|---|---|
| Pod UID matches notification | PASS |
| Booking container currently Ready; deployment 1/1 available | PASS at capture |
| Restart counter stable since latest recorded restart | PASS through 08:24:37; approximately 23m43s since current process start |
| `/health` via Kubernetes service proxy | PASS; returned `healthy` |
| `/health/ready` via Kubernetes service proxy | PASS; returned `ready`; deployed handler performs `SELECT 1` |
| Restart alert | INACTIVE at 08:22:46 evaluation |
| Durable application correction | NOT IMPLEMENTED; unsafe scheduler code remains deployed |
| Underlying dependency/network cause | UNRESOLVED |
| Booking creation/payment/expiry end-to-end validation | NOT TESTED |
| Historical data integrity and message reconciliation | NOT TESTED |

The health requests exercise the Kubernetes API service-proxy path, not a real client's ingress/authentication/booking path. Their success does not establish full product functionality.

**Corrective actions.** All actions below are proposed, unimplemented, and require assignment. Relative target dates begin when the incident owner accepts the plan.

| Priority / proposed owner / target | Action | Acceptance evidence |
|---|---|---|
| P1 / booking-service maintainer / next working day | Catch failures around the entire expiry task, including initial selection; add structured error fields, bounded retry/backoff, and a single-running-job guard. Keep failed work observable. | Dependency failure injection demonstrates no unhandled rejection, no process exit, bounded retry, continued local liveness, and accurate NotReady behavior while DB access fails. |
| P1 / platform + database owner / next working day | Correlate DNS, TCP reachability, service routing/CNI, host pressure, PostgreSQL connections/locks, and RabbitMQ health around retained failure windows. | A timestamped causal finding with supporting telemetry, or explicit elimination evidence for each tested hypothesis. |
| P1 / application QA + maintainer / before corrective rollout | Test DB outage/recovery, pool timeouts, concurrent payment/expiry transitions, and message redelivery. Review expiry idempotency and atomic state preconditions. | Deterministic tests show no duplicate cancellation/promo effects, no payment-state overwrite, and backlog recovery. |
| P2 / platform observability / within 3 working days | Add desired-versus-available replica alerting, request error/latency SLIs, expiry success/lag/failure metrics, and dependency reachability diagnostics. Preserve restart alert as an additional signal. | Tests demonstrate detection of readiness-only outages and scheduler failure without depending on process crashes. |
| P2 / application + platform / within 3 working days | Review readiness query time budget, pool limits, and connection budgeting across services; set the service name in structured logs. | Agreed budgets, load-test evidence, and logs attributed to booking-service. |
| P2 / application + platform / within 5 working days | Evaluate two replicas or isolating expiry in a worker after addressing concurrency and idempotency. | Failover works without duplicate expiry side effects; acknowledge shared dependencies can still affect all replicas. |
| P2 / platform / within 3 working days | Retain previous crash logs, deployment/image provenance, alert transitions, and request metrics beyond event expiry. | A repeat investigation can attribute each restart and quantify user-facing impact. |

Increasing memory, relaxing restart alerts, or merely restarting the deployment has no evidence-backed basis as a durable correction for the retained failure. Scaling alone would also introduce multiple expiry timers unless concurrency is addressed.

**Closure criteria.** Close corrective work only after the reviewed application change is deployed with recorded image identity, controlled dependency-failure tests pass, and recovery is demonstrated through booking, payment, expiry, and message processing checks. Proposed initial observation is at least 60 minutes with no unexpected restarts, healthy request/error SLIs, and successful scheduled work, followed by a 24-hour review because failures here recurred intermittently across more than seven hours. Observation duration is an acceptance proposal, not evidence already gathered. Any unresolved initiating connectivity cause must remain tracked separately rather than being concealed by the application resilience fix.

**What helped and what needs improvement.** Pod UID and immutable runtime image digest made exact attribution possible. Previous-container logs preserved the fatal error, live code exposed the missing error boundary, and retained metrics established recurrence beyond the latest crash. Kubernetes and AMQP retries restored service without intervention. However, a noncritical recurring task could terminate the entire service, the singleton had no replica redundancy, readiness failures extended beyond restart-alert coverage, and the retained evidence does not quantify customer or booking impact. These are system design and observability findings, not individual blame.

**Evidence and limitations.** See [public evidence index](evidence/README.md) for file mapping, query bounds, capture commands, redaction scope, and checksums. This report uses live evidence from this incident; no previous incident's diagnosis or recovery result was reused. Only the current and immediately previous booking process logs were captured, with explicit tail limits. PostgreSQL logs were limited to nine hours and 5,000 lines. Kubernetes events are retained aggregates rather than a complete event history. Loki history, packet captures, database workload inspection, host suspend records, notification delivery records, and end-user request traces were not collected. Those limits prevent a definitive initiating-infrastructure root cause or attribution of all 44 new restarts to the same failure mode.
