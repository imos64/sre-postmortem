# Corrective action register

[Incident guide](README.md) · [Full postmortem](2026-09-19-windows-wsl-runner-recovery-postmortem.md) · [Runbook](RUNBOOK.md)

Status is bounded to the completed incident evidence. **DONE** means implementation
and the listed acceptance were recorded. **OPEN — proposed** means recommended
follow-up, not an assigned external ticket or a promise that work was performed.
The platform owner must accept future ownership and scheduling. P1/P2 are action
priorities, not a retrospectively invented incident severity.

## Completed restoration and prevention

| ID | Priority | Action | Owner at execution | Status and acceptance |
| --- | --- | --- | --- | --- |
| C01 | P1 | Restore real paired runner instead of standby | Platform operator with repair automation | **DONE** — accepted poll at 21:51:20.264 UTC using existing pairing; unchanged original Runner source |
| C02 | P1 | Add durable runner and connector dependency gates/restart backoff outside the mutable app | Platform operator with repair automation | **DONE** — root-owned final drop-ins; isolated tests; idle runner failure recovered with accepted heartbeat in about 16 seconds |
| C03 | P1 | Supervise Windows launch, foreground WSL lifetime and Linux dependencies continuously | Platform operator with repair automation | **DONE** — retry/singleton cases and final real session 0 boot recovery before sign-in |
| C04 | P1 | Replace one-attempt helper's indefinite wait with bounded truthful observation | Platform operator with repair automation | **DONE** — helper tests and installed/source copies updated; task Running no longer accepted as health |
| C05 | P1 | Add application-level accepted-heartbeat and full platform health gates | Platform operator with repair automation | **DONE** — rejects stale/old-boot/PID mismatch; live final health, independent remote samples and owner Ready confirmation |
| C06 | P1 | Correct PowerShell default-argument defect introduced by the repair | Repair automation | **DONE** — move path resolution into script body; omitted-argument and actual File invocation coverage; final actual reboot passed |
| C07 | P1 | Correct Python import shadowing and preserve original execution/reconciliation semantics | Repair automation | **DONE** — isolated import regression, 11 runner tests and accepted live responses; no replay added |
| C08 | P1 | Repair related address/proxy assumptions without weakening identity/security guards | Platform operator with repair automation | **DONE within recorded scope** — preceding admission/network checks; exact API /32 and proxy-owner checks; final boot configuration/readiness validation |
| C09 | P1 | Preserve state and make rollback reviewable | Platform operator with repair automation | **DONE** — 55 storage pairs, 51 records and scoped 4,209 files verified; guarded Windows and 15-file Linux dry-runs passed |
| C10 | P1 | Prove actual automatic recovery before sign-in | Platform owner and repair automation | **DONE** — third reboot reached full readiness and sustained accepted heartbeats before recorded sign-in; first two limitations retained |
| C11 | P2 | Retire only completed qualification machinery | Repair automation | **DONE** — proof recorder archived/disabled; core recovery and keeper remained active; final health passed |

Evidence: [installed repair](evidence/installed-recovery.json),
[fault tests](evidence/failure-injection.json),
[regressions](evidence/regression-tests.json),
[reboot outcomes](evidence/reboot-outcomes.json),
[final health](evidence/final-health.json),
[preservation](evidence/preservation.json), and
[rollback verification](evidence/rollback-verification.json).

## Open follow-up proposals

| ID | Priority | Proposed work | Proposed owner | Scheduling gate | Measurable closure criterion |
| --- | --- | --- | --- | --- | --- |
| F01 | P1 | Qualify runner installer/interface compatibility | Platform owner / runner maintainer | Before the next runner update | Rehearse with a non-dispatch fixture or separate nonproduction pairing; never let a clone use production pairing. Verify installer/drop-in and reconciliation contracts, then confirm accepted heartbeat on the original runner during a planned idle update; no duplicate runner or replay |
| F02 | P1 | Add an external stale-heartbeat/recovery-failure notification | Platform owner / monitoring maintainer | Next monitoring maintenance window; owner to set date | An independently observed stale accepted heartbeat delivers a test notification, recovery resolves it, and UI Offline remains truthful |
| F03 | P2 | Review the two orderer headless warnings and intended consumers | Platform owner / Fabric maintainer | After consumer/dependency inventory | Either restore intended ready endpoints or explicitly retire unused services after review; update enrollment and verification without masking a live failure |
| F04 | P2 | Retain useful Windows task/session and Linux recovery history | Platform owner | Next Windows maintenance window | A test boot retains task start/exit/retry and session timing across restart with bounded retention, credential exclusion and readable capture procedure |
| F05 | P2 | Qualify sustained real dependency failure | Platform owner | Separately approved idle maintenance plan | Delayed actual engine/network recovery resumes the platform without duplicate work or identity drift; preserve complete before/after evidence |
| F06 | P2 | Define a recovery objective and collect repeated boot measurements | Platform owner | After an agreed service requirement | Document an accepted readiness objective, collect multiple real boot intervals, distinguish warm restart/cold boot and sign-in timing, and alert on missed recovery |
| F07 | P2 | Evaluate failure domains and restore capability | Platform owner | Separate design/backup review before disruptive work | Document single-host risks, restore a scoped backup in an isolated environment, measure recovery and verify application data; obtain approval for any migration |
| F08 | P2 | Automate publication/acceptance consistency checks | Platform owner / automation maintainer | Next runbook revision | Reject process-only evidence, stale boot markers, missing sign-in timeline and incorrect probe protocol totals; preserve rejected attempts |

No due dates or external owners were fabricated. Future high availability,
architectural migration, destructive restore, shared-network interruption and paid
trial execution are outside the completed repair. They require their own reviewed
scope. The current healthy state does not close these proposals automatically.
