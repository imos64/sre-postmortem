# Public evidence — Windows/WSL runner recovery

[Incident guide](../README.md) · [Full postmortem](../2026-09-19-windows-wsl-runner-recovery-postmortem.md)

This is a reviewed, bounded publication of retained September 19–20, 2026 incident
records. All timestamps are UTC. It is not a raw archive export. Publication used
read-only local files and did not perform new service operations, API job polls,
fault injections or health probes.

| Evidence | What it establishes |
| --- | --- |
| [Original failure](original-failure.json), [exact code/journal excerpts](original-code-and-journal-excerpts.txt) | Dormant standby override, absent authenticated heartbeats, Restart=no, one-attempt helper followed by sleep |
| [Related startup failures](related-startup-failures.json) | Separate Cloudflare/API address drift, proxy identity handling and gate timeout failures from earlier same-day repair |
| [Installed recovery](installed-recovery.json), [exact policy excerpts](installed-policy-excerpts.txt) | Boot versus sign-in launch, existing native Docker, ordered identity guards, retry and restart policies |
| [Failure injection](failure-injection.json) | Clean runner-only delayed dependency and verified idle process crash recovery; excluded fixture failure |
| [Regression results](regression-tests.json), [retained raw test output](regression-test-output.txt) | 21 supervisor and 11 runner raw results; 3 helper and 14 health/observer summary counts; isolated/mocked scopes |
| [All reboot outcomes](reboot-outcomes.json) | Failed first implementation, second early-sign-in limitation, final accepted third test |
| [Final before-sign-in proof](final-before-signin.json), [automatic Linux observations](automatic-observer.json), [Windows session observations](windows-session-observations.json) | Actual Windows boot and console logon times joined to automatic Linux readiness/heartbeat proof |
| [Final health](final-health.json), [endpoint breakdown](endpoint-verification.json) | Five Ready nodes, 131 controllers, 55 storage pairs, active services plus real heartbeats; corrected 10 HTTPS + 1 local HTTP count |
| [Accepted heartbeats](accepted-heartbeats.json) | Three independent advancing remote acceptance timestamps across 126.439 seconds after final boot |
| [Preservation](preservation.json) | Scoped 4,209-file checks and 51 existing record comparisons without publishing private contents/IDs |
| [Dashboard confirmation](dashboard-confirmation.json) | Owner's authenticated dashboard refresh reported Ready after final boot; owner attestation, not automated screenshot |
| [Rollback verification](rollback-verification.json) | Guarded dry-runs, scope and intentional return to original Offline standby behavior |
| [Provenance](PROVENANCE.json), [checksums](SHA256SUMS) | Exact private-source relative paths and SHA256 plus transformations; public byte integrity |

JSON files are allowlisted summaries or selected fields unless provenance explicitly
marks them unchanged. TXT excerpts identify exact source line ranges or retained
test output. Source hashes establish which private artifacts were consulted; they
do not expose individual credential hashes or make inaccessible private evidence
independently reproducible. No raw logs, task XML principal identities, machine
UUIDs, pairing identifiers, credential/provider values or hashes, kubeconfigs,
business row IDs/content, or private administrative URLs are published.

The final proof is a joined verdict: automatic Linux observations alone defer the
sign-in question to Windows session events. Task Running, systemctl active and a
successful manual post-boot command are insufficient acceptance. Windows session
samples and recorded logon events support the qualified final interval; they do
not imply comprehensive historical audit coverage. Earlier failed tests remain
explicit. Historical Task Scheduler Operational logging was disabled and
LSA/Operational unavailable. The exact intermediate power-off cause in test two
was not identified.

Two pre-existing headless-service endpoint warnings remain visible. No paid trial,
authenticated business-write transaction or broader database-integrity test was
run. The final coordinator security check covers configuration/readiness, not
packet qualification. File hashes cover files up to 1 MiB plus credential config
and original Runner; larger retained files were size-checked. Three helper and
fourteen combined health/observer passes and the Linux rollback dry-run are
supported here by final narrative receipts, not separate raw output files.

The private prose's “11 public HTTPS checks” wording is corrected here: the actual
retained final records contain ten HTTPS checks and one local loopback HTTP check,
all matching expected statuses (seven 200 and four 401).

From this directory:

```bash
sha256sum -c SHA256SUMS
```

Checksums cover every public evidence file except SHA256SUMS itself. They verify
published byte integrity, not independent proof of causality or completeness.
