# SRE Postmortem: Windows/WSL Startup Failure and Terminal-Bench Runner Offline

[Architecture](README.md) · [Evidence](evidence/README.md) · [Runbook and rollback](RUNBOOK.md) · [Action register](ACTIONS.md)

## Incident record

| Field | Recorded value |
| --- | --- |
| Incident ID | `RUNNER-2026-09-19` |
| Incident date | September 19, 2026 in America/New_York; UTC evidence continues into September 20 |
| Affected environment | NucBoxK10 Windows host; Ubuntu-24.04 under WSL2; native Docker; existing kind clusters `dev` and `imos-terminal-acceptance` |
| Primary user symptom | Terminal-Bench dashboard: `Runner Offline — Imos dedicated runner` after restart |
| Related symptom | Public application returned Cloudflare Tunnel error 1033; startup helper reported connector timeout |
| Final outcome | **PASS: automatic platform recovery before Windows sign-in demonstrated after an actual reboot** |
| Severity | Not formally assigned; no production-wide severity or affected-customer count was established |
| Response | Platform owner and repair automation; authorized local repairs, guarded idle testing and Windows elevation |
| Final health capture | 2026-09-20 03:24:46 UTC; `READY_WITH_WARNINGS`, runner `READY`, errors empty |
| Publication scope | Historical incident analysis and reviewed evidence; no new deployment or fault injection during publication |

## Executive summary

The original reboot launched Ubuntu and started a systemd unit named
`imos-benchmark-runner.service`, but that unit did not run the paired benchmark
runner. A September 17 drop-in replaced its command with `benchmark-standby.py`.
The standby held the runner lock and reported local readiness without polling the
control plane. The dashboard correctly showed Offline because no fresh runner
heartbeats were accepted.

The previously running real runner had masked the dormant override. An earlier
manual startup test found containers and services already active, so it never
exercised the command that a fresh service start would execute. The base service
also used `Restart=no`. The Windows sign-in helper attempted startup once and
could then remain alive indefinitely even after failure. These were independent
reasons why task Running or service active could not establish recovery.

The repair restored the existing paired runner and added continuous supervision
across Windows, WSL lifetime, dependencies, and Linux services. It preserved the
native Docker engine, clusters, storage, pairing, task packages and run records.
A root-owned wrapper and final service drop-ins sit outside the updateable runner
application. They retain the original execution protocol while adding accepted
heartbeat receipts, dependency gates, retry backoff and process restart recovery.

Qualification required three actual Windows boot tests. The first exposed a new
PowerShell default-argument defect introduced by the repair. The second recovered
automatically but full readiness followed the owner's sign-in, so it did not prove
the before-sign-in requirement. On the third boot, full readiness passed at
03:01:21 UTC and sustained-heartbeat proof completed at 03:04:38, before Windows
recorded sign-in at 03:11:24. The owner subsequently confirmed dashboard Ready.

This closes the requested automatic recovery incident for the tested installation.
It does not establish high availability, indefinite uptime, future update
compatibility, or successful execution of a new paid benchmark trial.

## Impact and measurement

The dedicated runner was unavailable to the control plane while the standby
process occupied its service and lock. The user saw Offline and could not rely on
that runner being ready for submissions. The initial safe queue inventory showed
no queued, executing, or cancellation-pending work, so diagnosis and fault testing
could proceed after fresh idle checks. No paid trials were automatically replayed.

The related public ingress incident produced HTTP 530 responses with Cloudflare
error 1033 across nine checked public routes. Those ingress and tunnel failures
were repaired earlier in the same response and verified again during final
platform qualification. Their cause should not be substituted for the runner's
confirmed standby-command defect.

| Measurement | Evidence-backed interpretation |
| --- | --- |
| Last known pre-reboot accepted heartbeat | 2026-09-19 21:08:22.992 UTC |
| First accepted poll after restoring the real runner | 2026-09-19 21:51:20.264 UTC |
| Gap between those observations | **42 minutes 57.272 seconds**; not an exact measured dashboard outage duration |
| Final boot to full verified readiness | **175.773 seconds**; one observed recovery interval, not an SLO percentile |
| Final boot to completed automatic proof | **373.494 seconds**; includes the required observation period |
| Proof completion before first sign-in | **405.506 seconds** |
| Idle process-failure recovery | About **16 seconds** from controlled SIGKILL to a new process's accepted poll |
| Preserved queue/history scope | 51 existing job/draft/refresh records with unchanged IDs, status and update times |
| Storage scope | 55 PVCs paired with 55 PVs; all enrolled backing directories and identities verified |

The exact moment the dashboard crossed its Offline threshold, notification
latency, failed submission count, and end-user transaction impact were not
measured. No formal availability SLO or error-budget calculation was available.
The complete investigation and planned reboot-testing period is not continuous
service downtime. Preservation checks found no changes within their stated scope;
they are not a universal proof of every application database's integrity.

## Detection and initial evidence

Detection came from the owner: the startup log reported a 60-second connector
`TimeoutExpired`, while the Terminal-Bench dashboard showed Offline. The helper's
later Running state and the runner's active unit state conflicted with the
service-level symptom. The correct next step was to inspect the effective command
and accepted heartbeat history rather than dismiss the dashboard.

Before changing the installed runner, the investigation preserved prior/current
boot journals, the base unit, effective drop-in, standby source, selected runner
configuration metadata, the last accepted control-plane heartbeat, and a safe
queue inventory. Windows task definitions, ACLs, execution capability, WSL
registration, native Docker identity and startup logs were also retained.
Credentials, provider keys and pairing secrets were excluded from terminal output
and reports. The public bundle contains selected excerpts and allowlisted summaries;
private configuration and unrestricted archives were not uploaded.

The dashboard's deployed frontend showed that Offline follows `runner.online`;
Ready additionally requires backend readiness. A successful local process check
could not satisfy either application-level requirement. Early unauthenticated
owner-API attempts did not prove dashboard state. Final acceptance instead combined
real successful runner responses, independent accepted-heartbeat observations,
and an authenticated dashboard refresh confirmed by the owner.

See [original failure](evidence/original-failure.json) and
[selected original source/journal excerpts](evidence/original-code-and-journal-excerpts.txt).

## Actual architecture and startup responsibility

The enrolled Docker context was `default`, connected to the native Ubuntu daemon
at `/var/run/docker.sock`. Docker Desktop was a separate installation and engine.
The existing clusters already belonged to native Docker, so the repair did not
migrate engines, recreate kind, move volumes, or enable automatic Windows login.

Two Windows tasks served different roles. The user-named
`Imos-Control-Planes-77ff47b974` was the sign-in helper task. The existing
`TreeTracker kind-dev startup` task already had an AtStartup trigger with a
30-second delay and an S4U principal. Its Action was updated to the reviewed
Windows-local retry wrapper, preserving its trigger, principal, settings and DACL.
The final test observed it running in Windows session 0 before interactive sign-in.

Docker documents its Desktop startup option as sign-in-triggered. Microsoft
separately documents that systemd manages services within WSL and that systemd
services alone do not keep a WSL instance alive. These explain why a boot trigger,
a foreground WSL keeper, and Linux supervision had to be verified separately.
The successful final test establishes unattended startup for this installation;
it does not imply that every Docker Desktop setup supports the same behavior.
[Docker settings](https://docs.docker.com/desktop/settings-and-maintenance/settings/),
[WSL systemd documentation](https://learn.microsoft.com/en-us/windows/wsl/systemd).

## Causal analysis

### Primary failure: the service name concealed a different command

1. The real paired runner was already active before the original reboot.
2. An existing drop-in had replaced the next startup command with a standby
   process; that change was dormant until service restart.
3. The reboot stopped the real runner at 21:08:31 UTC.
4. At 21:10:33, systemd started the standby, which logged `READY_NO_DISPATCH`,
   `dispatch_enabled=false`, and `remote_queue_polled=false`.
5. The standby held the normal runner lock but never sent authenticated polls or
   heartbeats. The control plane's last-seen value stopped advancing.
6. The service remained active, while the dashboard correctly indicated Offline.

The evidence supports an installed-command/policy mismatch, not an expired pairing
secret or an authentication rejection. Restoring the original paired application
produced accepted responses without rotating credentials. The earlier standby
policy had been documented as not providing remote executor presence; the failure
was treating it as satisfying the later automatic runner recovery requirement.
No individual intent or undisclosed change is inferred from the evidence.

### Contributing recovery and detection gaps

| Condition | Effect | Repair or acceptance response |
| --- | --- | --- |
| Base runner `Restart=no` | An exited real runner would remain offline | Persistent restart policy with capped backoff |
| Helper made one attempt then waited indefinitely | A Running task could survive failed startup without retrying recovery | Continuous boot/WSL/Linux supervision; bounded sign-in observer |
| Already-active manual startup test | Did not exercise the effective fresh-start command or boot ordering | Actual reboot tests with pre-sign-in observation |
| Dependency and launcher timeouts disagreed | A 60-second wrapper timeout could expire during a legitimate 420-second gate | Earlier asynchronous service activation with an explicit 450-second monitor, followed by durable supervision |
| WSL lifetime inferred from service state | Active Linux services did not demonstrate a persistent foreground WSL session | Held keeper lock and current Windows/WSL marker required by health |
| Process liveness used as readiness | Standby looked healthy locally | Fresh accepted heartbeat, matching PID/boot and functional platform checks |
| Single-host dependency chain | A host restart affects the whole enrolled platform | Measured recovery; high availability remains separate future work |

### Related Cloudflare, ingress and coordinator failures

The preceding startup repair identified three additional boot-sensitive assumptions:
a tunnel init guard used hardcoded worker addresses; an ingress policy allowed a
stale Kubernetes API address; and the startup coordinator treated absent,
controller-owned stateless proxy containers like missing persistent node/data
containers. Those conditions prevented tunnel and ingress recovery and blocked
coordinator progress.

The repairs used stable node identity plus a validated current host address,
refreshed only the verified API endpoint's exact `/32`, and verified proxy owners
before allowing the existing controller to reconcile its proxies. Persistent
container, cluster and storage identity guards remained in place. There was no
broad firewall exception, replacement tunnel, DNS change, or credential rotation.
Cloudflare's description of 1033 is consistent with a missing healthy tunnel
connector; it does not identify which local dependency failed.
[Cloudflare error documentation](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/troubleshoot-tunnels/common-errors/#i-see-an-error-1033-when-attempting-to-run-a-tunnel).

The preceding incident reached coordinator readiness at 21:28:28 UTC and repeated
public checks at 21:29. It explicitly did not perform a fresh reboot. Its live
admission and tunnel-network probes are historical checks from that earlier phase;
the final reboot reused configuration/readiness verification and did not repeat a
new packet-level security qualification. See
[related startup evidence](evidence/related-startup-failures.json).

### Defects discovered while implementing the repair

The repair introduced a PowerShell default-value bug. Under the task's real
`powershell.exe -File` invocation, `$PSScriptRoot` was empty during evaluation of a
parameter default used to construct the log directory. The wrapper failed before
logging or launching its keeper. Earlier mocked tests supplied an explicit log
directory, so they bypassed the failing default expression. This was a defect in
the repair and its test coverage, not a successful reboot or a user error.

The default path resolution was moved into the script body, and the exact omitted
argument invocation was added to regression coverage. The first failed reboot
remains marked NOT_ACCEPTED. During that rejected test, the Linux coordinator
detected an enrolled API-endpoint readiness mismatch, retried automatically, and
passed full verification around 22:16:22–23 before the later WSL shutdown. That
real retry was useful evidence, but did not prove continuous unattended recovery. The implementation also encountered Python import
shadowing from a nearby `platform.py`; isolated Python execution and an integration
regression corrected that before final qualification.

## Response timeline

All times are UTC. September 20 UTC still falls on September 19 in the owner's
America/New_York timezone. The first-test directory name records its reboot request
time, not a separately inferred Windows boot timestamp.

| Time | Observation or action | Meaning |
| --- | --- | --- |
| Sep 19 21:08:22.992 | Last pre-reboot accepted heartbeat | Paired runner authenticated before restart |
| 21:08:31 | Real runner stopped, preserving pending reports | Fresh startup would exercise installed override |
| 21:08:58.500 / 21:09:12 | Original failed Windows boot / console sign-in | Both occurred before the reported helper and service starts |
| 21:09:32–21:10:35 | Startup attempt, container starts, connector timeout, helper continued waiting | Attempted startup did occur; Running was insufficient |
| 21:10:33 | Standby command started in runner service | Confirmed source of missing runner heartbeats |
| 21:14:48 | Timestamp on owner-supplied Cloudflare 1033 error page | Separate public ingress symptom; not a measured report-delivery time |
| 21:28:28–21:29 | Earlier tunnel/ingress/coordinator repair passed and public checks repeated | Current-session recovery; reboot untested at that stage |
| 21:51:20.264 | First accepted poll from restored real runner | Authentication recovered with existing pairing |
| 21:56:05.534 | Runner-only delayed Docker test released | Same process subsequently recovered |
| 21:56:35.803 | Accepted poll after delay | Backoff recovered without shared Docker/network restart |
| 21:56:51.236 | Idle runner deliberately killed | Controlled process-failure test after idle checks |
| 21:57:07.074 | Replacement process sent accepted poll | Approximately 16-second recovery |
| Before first reboot | Transient control-plane 503 postponed guarded reboot | Same process recovered; failed preflight retained |
| 22:11:59 / 22:13:29.500 | First reboot requested / actual Windows boot | Test later rejected because the new wrapper failed |
| 22:17:15–22:18:19 | WSL terminated and relaunched under the same kernel boot ID | Kernel boot ID alone could not establish WSL lifetime |
| 22:27:43 | Corrected S4U task manually started | Keeper check passed; explicitly excluded from reboot proof |
| Sep 20 01:53:26.500 | Second test's final Windows boot | Automatic startup observed |
| 01:54:13.048 | Windows recorded console sign-in | Occurred before full readiness |
| 01:56:35.236 | Full platform readiness | Automatic recovery passed, pre-sign-in requirement unproven |
| 01:59:44.195 | Second automatic observer completed | 10 accepted timestamps over 157.68 seconds |
| 02:58:25.500 | Third Windows boot | Final qualification began |
| 02:59:07.619 / 02:59:20.831 | Boot wrapper / foreground keeper started | Session 0 automatic startup path |
| 02:59:51 | Real runner started | Same process remained stable during final verification |
| 03:01:21.272672 | Full coordinator passed | Platform ready before sign-in |
| 03:04:38.994398 | Automatic proof completed | 12 accepted timestamps across 180.21 seconds |
| Through 03:09:12 | Windows observers accumulated 121 known signed-out samples | Samples continued after Linux proof completion |
| 03:11:24.500140 | First Windows console sign-in | More than six minutes after completed proof |
| 03:13:51 | First post-boot investigator read | Acceptance evidence already existed |
| 03:14:50–03:16:56 | Three independent control-plane observations | Fresh accepted heartbeats continued over 126.439 seconds |
| 03:18:01 | Guarded Windows rollback dry-run passed | No rollback applied |
| 03:19:17 | Final verdict saved; completed recorder archived and disabled | Core recovery supervision remained active |
| 03:24:46 | Final health snapshot | 107 successful polls, heartbeat age 8.76 seconds, no errors |

The second test included an intermediate restart/power-off sequence whose cause
was not determined. It is not attributed to Windows Update or automatic sign-in
policy without evidence. The owner confirmed signing in before the requested
interval; the result was therefore classified honestly and the test repeated.

## Implemented recovery design

| Layer | Installed behavior | Preservation and failure handling |
| --- | --- | --- |
| Windows boot Action | Windows-local `startup-supervised.ps1`; WSL launch retries from 10 to 300 seconds, with a 300-second initialization timeout | Existing task trigger/principal/settings/DACL preserved; no distro reset or forced shutdown |
| WSL lifetime | Singleton foreground `imos-platform-wsl-session`; current Windows/WSL marker; ensure recovery supervisor | Held lock checked independently of process existence |
| Linux recovery | Root `imos-platform-recovery.service` and `supervise.py`; engine and enrolled identity guards; networking/nodes before dependent services | Start inactive dependencies; do not restart active workloads merely because a probe runs; persistent retries |
| Full coordinator | Repeatable oneshot with `RemainAfterExit=no`; real heartbeat acceptance; full verification periodically and after relevant recovery changes | Existing cluster/data identity checks retained; full verification cadence six hours |
| Runner | Root-owned wrapper dynamically loads unchanged installed Runner; final `zzz-recovery.conf`; live node gate; restart delay 10–60 seconds without permanent rate-limit lockout | Original lease/execution/report protocol retained; no separate replay mechanism |
| Connector | Root-owned final drop-in with live node gate and persistent restart backoff | Existing connector and pairing retained |
| Sign-in helper | Bounded observation with truthful exit status | Task Running no longer used as proof of health or WSL lifetime |
| Health | Live Docker, cluster/node/workload/storage checks; accepted heartbeat within 45 seconds; matching boot/PID; fresh supervisor within 120 seconds; held keeper lock | No job polling or dispatch from the health executable; stale receipts fail |

The runner and connector execute as Linux user `imos`; root ownership protects
the supervision files and does not mean the runner executes as root.

The root-owned files survive the current package installer's replacement of the
base user service and runner application directory. This is update isolation, not
proof that arbitrary future Runner API or installer changes cannot break the
integration. Upgrade-contract qualification remains an explicit follow-up.

The runner wrapper archives interrupted-run records before invoking the installed
runner's original reconciliation. It does not delete locks or replay saved work.
The original application source remained unchanged. Windows Action installation
required the owner's approved UAC elevation after non-elevated updates were denied;
failed/canceled attempts were retained. The owner chose to keep the current sign-in
setting, and no ARSO policy modification was made.

See [installed recovery](evidence/installed-recovery.json),
[policy excerpts](evidence/installed-policy-excerpts.txt), and the
[operations runbook](RUNBOOK.md).

## Verification and acceptance

### Three actual boot tests

| Test | Result | Acceptance reason |
| --- | --- | --- |
| First, reboot requested Sep 19 22:11:59 | **NOT_ACCEPTED** | New wrapper default expression failed before keeper launch; sign-in recovery and a later manual task start did not qualify |
| Second, final boot Sep 20 01:53:26.500 | **PASS automatic recovery; before-sign-in NOT_PROVEN** | Full readiness at 01:56:35 followed console sign-in at 01:54:13 |
| Third, boot Sep 20 02:58:25.500 | **PASS before sign-in** | Full readiness and sustained accepted-heartbeat proof preceded console sign-in; one continuous wrapper/keeper; no manual startup commands |

For the final test, the owner signed out and restarted from the Windows sign-in
screen. A visible lock screen alone was not accepted as evidence of no signed-in
session. Windows session observations and events were compared with the automatic
Linux proof. Microsoft documents that automatic restart sign-in depends on the
last interactive user not having signed out before restart; no policy change was
needed for this executed test.
[Microsoft ARSO documentation](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/component-updates/winlogon-automatic-restart-sign-on--arso-).

The Linux observer ran for 309.77 seconds and recorded 12 accepted heartbeat
timestamps spanning 180.21 seconds. The wrapper and independent Windows observer
together recorded 121 known signed-out samples over an approximately 604.80-second
span through 03:09:12. These are distinct observation windows, not 121 samples all
collected by the earlier 03:04:38 proof completion.

Later independent control-plane reads observed accepted `last_seen` values of
03:14:44.060, 03:15:40.990 and 03:16:51.410. Observation times spanned 126.439 seconds.
The runner PID was stable with zero service restarts, and the owner refreshed the
authenticated dashboard and confirmed Ready. These checks establish real accepted
heartbeats rather than attempted requests or a locally invented online flag.
[Reboot outcomes](evidence/reboot-outcomes.json),
[final before-sign-in proof](evidence/final-before-signin.json),
[accepted heartbeat samples](evidence/accepted-heartbeats.json).

### Fault tests and final platform checks

| Check | Result | Boundary |
| --- | --- | --- |
| Delayed runner dependency | **PASS**: a runner-only Docker shim failed two doctor checks; same process recovered after release | Shared Docker/network were not stopped; not a physical WAN outage test |
| Idle process failure | **PASS**: SIGKILL followed by systemd restart and accepted heartbeat in about 16 seconds | Performed only after fresh local/remote idle checks; no active benchmark killed |
| Isolated regression tests | **PASS**: 21 supervisor, 11 runner, 3 helper, 14 health/observer tests; additional isolated systemd and Windows-wrapper cases | Includes corrected default invocation/import isolation; unit tests alone were not boot proof |
| Docker and nodes | **PASS**: enrolled native engine; 4 dev nodes and 1 acceptance node Ready with matching identities | Existing containers/clusters retained |
| Workloads | **PASS**: 123 dev plus 8 acceptance controllers verified | Two existing unmatched headless endpoint warnings remained visible |
| Storage | **PASS**: 55 PVC/PV pairs and backing directories verified | Identity/presence checks are not a destructive restore test |
| Functional platform | **PASS**: read-only database checks, eight peer memberships/channel queries, application ledger reads and orderer TLS | No business write transaction or new paid trial |
| Endpoint checks | **PASS**: 11/11 total; **10 HTTPS plus 1 local HTTP** health check | Seven HTTP 200 and four expected protected-route 401 responses |
| Runner/control plane/dashboard | **PASS**: accepted responses, independent advancing last-seen, owner-confirmed Ready | No weaker Offline predicate or authentication bypass |
| Rollback | **PASS dry-runs**: Windows Action guard and 15 Linux file guards | Rollback was not applied |

The endpoint count corrects an imprecise private summary that called all eleven
checks HTTPS. The underlying final result contains ten HTTPS routes and a local
loopback MinIO HTTP health check. It does not show eleven public HTTPS requests.
This report follows the retained endpoint records; original private evidence was
not silently rewritten. See [endpoint verification](evidence/endpoint-verification.json).

## Preservation and safety

No clusters were recreated, Docker resources pruned, provider credentials rotated,
pairing reset, locks blindly removed, or paid benchmark trials automatically
replayed. The interrupted-record wrapper preserves the original recovery protocol
and archives input records before reconciliation.

The final 51-record comparison covered 40 terminal Terminal-Bench jobs, eight Harbor
records that were terminal or validated, two review/approved drafts, and one failed
refresh record. Their IDs, status and update timestamps were unchanged. No active
jobs, new trials, added records or removed records appeared in that scope. This
is not a byte comparison of the entire remote database; unrelated session and
capability metadata can change independently.

File preservation passed for 4,209 retained files. Files up to 1 MiB were hashed,
as were the credential configuration and original Runner source; larger retained
files were size-checked. All 55 PVC/PV pairs and enrolled storage backing identities
were verified. A size check is weaker than a content hash, and neither this check
nor record parity establishes full application-level transaction correctness.
See [preservation evidence](evidence/preservation.json).

Reboot and failure injection were gated on idle workloads. A transient control-plane
503 postponed a planned reboot until the same runner recovered. The final recorder
was archived and disabled after successful qualification; continuous recovery and
the foreground keeper remained active. No further reboot was left scheduled by
this repair.

## What worked, what failed, and lessons

**What worked:** the dashboard retained a truthful Offline indicator; original
pairing and application code were reusable; effective-unit inspection explained
the discrepancy; identity guards protected persistent resources; idle checks
prevented disruptive testing; automatic pre-sign-in recording survived investigator
disconnection; independent remote samples and owner confirmation completed the
service-level evidence chain.

**What failed:** manual checks of an already-running system did not exercise boot;
standby state was confused with executor readiness; a helper could outlive its only
startup attempt; the first mocked Windows test omitted the default invocation;
WSL lifetime and kernel boot identity were initially insufficiently distinguished.
The first actual reboot exposed a defect introduced by the repair, and the second
test did not meet the sign-in timing condition. Both results remain visible.

**Lessons:** define health at the service boundary, inspect the effective command
that will run next, supervise every lifecycle boundary, test the actual executable
invocation, and preserve evidence before any command that might start a stopped
dependency. A passing reboot proves one installation and one observed sequence;
repeatable qualification and update checks are still required for ongoing assurance.

## Remaining risks and corrective work

The completed and proposed actions, owners, priorities and measurable closure
criteria are maintained in [ACTIONS.md](ACTIONS.md). Proposed work is not presented
as implemented or assigned to another person. No external issues or notifications
were created as a side effect of writing this postmortem.

The two existing warnings are `hlf-orderer/raft-orderer-headless` and
`hlf-orderer/v2-raft-orderer-headless`. They remained visible in `READY_WITH_WARNINGS`;
resolving or retiring them requires a separate review of their intended consumers.
Future runner updates need contract checks, and the single host remains a failure
domain. A future platform migration or disruptive full-engine/network fault test
requires its own concrete plan and workload-safe authorization.

## Evidence limitations and references

Historical scheduled-task history was disabled and LSA/Operational was unavailable.
The investigation did not read credential caches to compensate. Some precise
notification, API/UI transition and intermediate power-off causes therefore remain
unknown. No exact overall request downtime, production customer impact, paid-trial
completion, destructive restore result, or future-version compatibility is claimed.

The public [evidence index](evidence/README.md) distinguishes selected exact
excerpts, allowlisted JSON observations, and investigator-derived summaries.
[PROVENANCE.json](evidence/PROVENANCE.json) records source identities and documented
transformations; [SHA256SUMS](evidence/SHA256SUMS) checks published byte integrity.
Private archives remain local and are intentionally not uploaded wholesale.
The original full coordinator security/readiness checks and earlier packet probes
are different phases; neither is silently promoted into a later test result.

Upstream behavior was checked against the official Docker, Microsoft WSL/ARSO and
Cloudflare documentation linked above. Local evidence establishes this incident's
causes and outcomes; those general documents do not by themselves prove recovery.
