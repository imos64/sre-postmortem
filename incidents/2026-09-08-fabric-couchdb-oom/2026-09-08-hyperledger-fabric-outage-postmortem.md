# Hyperledger Fabric Development Outage — SRE Postmortem (2026-09-08)

[Incident guide and high-level architecture](README.md) · [All postmortems](../../README.md)

> Public edition: selected evidence is included in [evidence/](evidence/README.md). The raw archive is private. Upstream fix and CI links require access to the original private repositories.

| Field | Value |
| --- | --- |
| Incident | FABRIC-2026-09-08 |
| Environment | Local `kind-dev`, Kubernetes v1.36.1 |
| Affected namespaces | `hlf-peer-org`, `hlf-explorer` |
| Status | Recovered; final verification September 8, 2026 at 13:43 UTC / 09:43 EDT |
| Impact classification | Development Fabric network outage; production impact not established |
| Primary failure | Eight CouchDB containers repeatedly OOM-killed during Erlang startup |
| Secondary recovery blocker | Explorer's legacy signing identity and orderer TLS trust were stale; its wallet cached the stale identity |
| Resolution | Bounded Erlang port allocation; corrected Explorer profile and automatic wallet refresh |

## Summary and Impact

All eight CouchDB world-state databases and all eight Fabric peers were unavailable at investigation time. The user supplied a subset of failing Pods; the live audit also found the omitted Investor peers unavailable. Explorer was unable to initialize against Fabric and repeatedly exited. The eight databases, eight peers and one Explorer instance formed the 17 affected application Pods.

The five CA Pods, ten orderer Pods, two chaincode services and Explorer PostgreSQL were running during the initial audit. Their Pod health did not make peer-backed operations available. Operations that needed unavailable peer endorsement, query or ledger access could not complete; Explorer visualization/index updates were interrupted. No transaction-loss count or external-user impact was established.

The immediate failure was not insufficient node RAM. Nodes were using approximately 11–13% of their reported memory while individual CouchDB processes hit a 512 MiB cgroup limit. Their inherited `RLIMIT_NOFILE` was 1,073,741,816. Erlang sized startup data structures against this extreme limit and exceeded the container ceiling before normal application logging.

Restoring CouchDB recovered the peers but exposed an independent Explorer problem. Its legacy profile still used a historical peer identity and historical orderer CA bundle. Current channel policy rejected that identity; current orderer TLS certificates did not validate against the old bundle. Explorer also reused its persisted wallet entry even when source credentials changed. Both blockers were repaired without changing channel policy or minting new identities.

## Timeline

Times are UTC; EDT is UTC minus four hours.

| Time | Evidence / action |
| --- | --- |
| Sep 7 21:30:56 UTC / 17:30:56 EDT | Both kind worker containers' `StartedAt` timestamps. This is a plausible triggering restart, not a proven exact outage start. |
| Sep 8 13:19–13:27 UTC | Retained logs show peer CouchDB connection refusal, Explorer startup failure, and CouchDB cgroup OOM kills. Initial live audit confirmed the broader 17-Pod impact. |
| Approximately 13:28 UTC | Created isolated diagnostic Pod with the same CouchDB image and 512 MiB limit, no data volumes. Confirmed extreme inherited open-file limit. |
| Approximately 13:29 UTC | A/B test: Erlang with `+Q 65536` started; identical command without the cap exited 137 and OOM-killed the diagnostic Pod. |
| 13:30 UTC | Published peer GitOps commit `fe1b20d`; Argo reconciled all eight CouchDB Deployments. CouchDB logged successful startup by 13:30:25 UTC. |
| Approximately 13:31 UTC | Reset five peers still in backoff; others recovered through normal retries. No PVCs or ledger files were removed. |
| Approximately 13:32 UTC | All eight peers became ready. Five current-channel peers agreed at height 13. Restarted Explorer; its legacy identity/trust failure became visible after connectivity recovered. |
| Approximately 13:34–13:35 UTC | Existing current-root Explorer client successfully queried legacy height 22. Historical orderer trust failed TLS verification; current trust passed. Confirmed Explorer's wallet cache behavior in its installed source. |
| 2026-09-08T13:37:57Z | Replacement Explorer container started after GitOps commit `3e349bf`. Wallet init refreshed the legacy entry. |
| 13:40 UTC | Authenticated CouchDB health checks all passed; peers remained ready, Explorer index matched both ledgers, UI returned HTTP 200, and both Argo applications and CI runs were healthy. |

Exact outage onset and full customer downtime cannot be reconstructed from the retained evidence. Do not equate Pod age or restart count with downtime. The worker restart suggests a prolonged outage, but no time-series availability record was retrieved to prove its beginning. The exact investigation-start timestamp was not separately captured before the initial commands; no exact MTTR is claimed.

## Root Cause and Contributing Factors

### 1. CouchDB / Erlang startup allocation

- All eight original CouchDB containers terminated with `OOMKilled`, exit 137, approximately one second after starting.
- Kernel records identified `beam.smp` in the affected Pod cgroups at roughly 512 MiB resident memory, against the 512 MiB limit.
- Diagnostic `ulimit -n` and `/proc/self/limits` showed soft and hard limits of 1,073,741,816.
- The same image's Erlang runtime, using `start_clean` and `+Q 65536`, reported `port_limit=65536`, `memory=23791736` bytes and 20 schedulers, then exited successfully.
- Removing only `+Q 65536` reproduced exit 137/OOM under the same container limit. This isolated port-table sizing from production data corruption, scheduler tuning and memory-limit changes.

The high inherited descriptor limit is confirmed. The actor or host configuration change that introduced it is not established. The worker-container restart is temporally consistent with exposing the latent runtime dependency. Raising memory or repeatedly restarting unchanged Pods would not remove that dependency.

Apache documents this startup allocation behavior and recommends a bounded descriptor or Erlang port limit: [CouchDB startup-memory troubleshooting](https://docs.couchdb.org/en/stable/install/troubleshooting.html#lots-of-memory-being-used-on-startup).

### 2. Dependency cascade

Peers could not initialize their ledger provider while their individual CouchDB service refused connections. Their startup retries exhausted and the peer process panicked. Explorer then failed its default peer/channel query and exited. Explorer's exit code was 0 despite its fatal initialization error, so `CrashLoopBackOff` was not evidence of an application OOM.

### 3. Explorer identity, TLS and wallet drift

After peer recovery, Explorer's historical signing identity was denied on `treetracker`. Its old orderer bundle failed verification of the running orderer certificate. These were independent of CouchDB memory use and remained after network connectivity returned.

The already-provisioned `OU=client`, `CN=treetracker-v2-explorer` identity successfully queried both intended channel contexts; a legacy query returned the same height/hash as Greenstand peers. The current orderer bundle validated the legacy endpoint with TLS verification enabled. No channel ACL, root governance rule, or TLS verification setting was weakened.

Installed Explorer 2.0 code checked for an existing wallet entry and skipped loading mounted credentials when one existed. Persisted wallet state therefore needed reconciliation as well as connection-profile correction.

## Corrective Changes

1. Added `ERL_FLAGS="+Q 65536"` to all eight CouchDB Deployments. Preserved image digest, 512 MiB limits, CPU settings, security contexts, data claims and single-writer `Recreate` strategy. GitOps commit: [fe1b20d](https://github.com/HLF-Enterprise-Blockchain/hlf-peer-org/commit/fe1b20d30f20a32b5b4e36c76c00da6a802b4712).
2. Added a regression check covering both `kind-dev` and production renders. It requires the cap on all eight databases and rejects a deliberately removed cap. Only the authorized kind-dev application was reconciled.
3. Reset five stalled peer Pods after database recovery. Three recovered through existing retries. Restarted Explorer after its default peer became ready.
4. Pointed Explorer's legacy profile at its existing current-root client and current orderer trust. Preserved network/channel names, UI login identity and password, signing Secrets and database contents.
5. Added a pre-start wallet reconciler: validates certificate/private-key agreement for every configured identity before writes, updates stale entries atomically, enforces mode 0600, and leaves matching entries unchanged. The app starts only after this init container succeeds. Commit: [3e349bf](https://github.com/HLF-Enterprise-Blockchain/hlf-explorer/commit/3e349bfd8f52af8f634a2e75ddf4406db3ad19c3).
6. Added tests for stale refresh, idempotence, permissions, key mismatch without overwrite, and corrected legacy trust paths.
7. Removed the two incident diagnostic Pods and the superseded, unreferenced live Explorer template ConfigMap after retaining a local copy. The historical signing Secret remains available for recovery; no PVC, ledger, CA database or wallet volume was deleted.

## Recovery Verification

| Check | Result |
| --- | --- |
| CouchDB | 8/8 ready; authenticated `/_up` returned `status=ok` for all eight |
| CouchDB memory | Approximately 84–103 MiB observed after recovery, versus unchanged 512 MiB ceilings |
| CouchDB restarts | Zero on the replacement database Pods through verification |
| Fabric peers | 8/8 ready |
| Current channel | All five participating peers at `treetracker-v2` height 13, identical current and previous block hashes |
| Legacy channel | Three Greenstand peers at `treetracker` height 22 with identical hashes |
| Nonparticipating peers | Two Investor peers and one Verifier peer remain intentionally channel-less |
| Explorer | Ready, replacement container without restarts, ingress HTTP 200 |
| Explorer index | Legacy 22 blocks/22 transactions; v2 13 blocks/13 transactions; maximum indexed block numbers 21 and 12 respectively |
| Indexed chaincodes | `tree-contract` and `token-contract`, version 1.0, in both profiles |
| GitOps | Peer and Explorer applications `Synced/Healthy` at their corrective revisions |
| CI | [Peer regression workflow](https://github.com/HLF-Enterprise-Blockchain/hlf-peer-org/actions/runs/34232370232) and [Explorer workflow](https://github.com/HLF-Enterprise-Blockchain/hlf-explorer/actions/runs/34233156548) succeeded |

Current-channel hash: `ZfG2dHqJ9vunlBIrJA4/5BGfDLM5Ei5iYWKJGu7aSg0=`. Legacy Greenstand hash: `hfwtXYmjUHmFrgAQDm0NScOMnEXS62T61VztKmo9d74=`.

No ledger divergence was observed in the successful comparisons. This is not a proof that no client requests were lost during the outage. No synthetic business write, token transfer, CA enrollment or governance transaction was submitted. A peer-identity `_lifecycle` query was rejected by Writers policy; Explorer's authorized client successfully indexed the committed definitions. The recovery did not broaden that policy.

Final stability check at 13:43:34 UTC confirmed all 17 affected workloads ready, no restart-count increases since the 13:40 snapshot, and Explorer over five minutes old with zero restarts. Both Argo applications remained Synced/Healthy and the UI still returned HTTP 200. See `final-verification.json`.

## What Went Well

- Termination reasons, kernel evidence and an isolated A/B test identified the startup allocation without altering persisted state.
- Existing PVC node affinity retained databases on their backing workers during replacement.
- Argo self-healing reconciled the corrective source commits quickly; fixes survive another Pod restart.
- Layered verification revealed Explorer's second blocker instead of closing the incident on peer readiness alone.

## Gaps and Follow-up Actions

Owners below are proposed roles, not assignments sent to individuals.

| Priority | Action | Proposed owner | Status / acceptance criterion |
| --- | --- | --- | --- |
| P0 | Bound CouchDB startup allocation and protect it in CI | Fabric platform maintainer | Completed; eight rendered deployments and runtime health verified |
| P0 | Align Explorer identity/trust and refresh wallet on startup | Explorer maintainer | Completed; both channels indexed and mismatch regression rejected |
| P1 | Establish a controlled `nofile` default for the container runtime and test a worker restart | Cluster SRE | Open; test outside business hours, retain per-application cap, verify all relevant runtimes after restart |
| P1 | Audit OOM/CrashLoop and peer/Explorer availability alert routing and response | Observability SRE | Open; demonstrate a delivered alert and recovery notification; this investigation did not establish prior alert delivery |
| P1 | Add startup readiness gates that query the intended channel with the actual client identity and verify endpoint trust | Fabric/Explorer maintainers | Open; prevent a TCP listener from being the sole functional-health signal |
| P1 | Test identity rotation through Secret update, wallet refresh and real channel query | Identity maintainer | Open; current reconciler runs on startup, so Secret rotation still requires a controlled restart |
| P1 | Refresh and restore-test recovery snapshots containing both corrective commits | Recovery maintainer | Open; source is persisted in GitOps, but no off-cluster recovery snapshot refresh was performed in this incident |
| P2 | Resolve frozen legacy CBO governance/trust debt through the documented authorized process | Fabric governance owners | Open; current CBO identities still fail legacy Readers policy; do not bypass channel policy |

The orderer application's pre-existing `OutOfSync` state was observed but not altered. It was not the cause of the reported database/peer crash loops. Investigate its drift separately. The cluster still uses node-local storage despite its `do-block-storage` class name; successful Pod recovery is not protection from kind-node or host loss.

## Evidence and Rollback

The original evidence archive is retained privately by the incident owner. This public repository contains the postmortem and [selected evidence](evidence/README.md). The appendix inventories the private archive, documents its limits, and provides selected log excerpts without credentials.

Reverting the CouchDB cap without first correcting the inherited runtime limit recreates the OOM. Reverting Explorer to the historical identity/trust recreates the observed authorization/TLS failures. Prefer restoring the proven configuration. If a future capacity change requires a different port cap, test it under the actual cgroup memory limit and workload before promotion. Restoring a prior wallet must use a channel-authorized identity; the historical key's mere presence does not make it valid under current policy.


## Evidence Appendix — Archival Collection

The archival collection was assembled on September 10, 2026 UTC (September 9 EDT). This was a review of retained files, historical session output, Git objects and GitHub Actions records. No Kubernetes or Docker access was performed, and no new runtime results were substituted for the September 8 recovery evidence.

### Archive and Integrity

- **Private archive:** `FABRIC-2026-09-08-20260910T010326Z.zip`, retained privately by the incident owner; not downloadable from this repository.
- **Size:** 573,673 bytes; **195 files** inside the archive.
- **SHA-256:** `dbb8222af8ba464a6913c94c36b73990d7a2b74b7c5788f2676804b97053e07b`.
- **Navigation:** the extracted bundle contains `README.md`, `EVIDENCE-MAP.md` and `COMMAND-LOG-INDEX.md`.
- **Provenance:** `SOURCES.json` maps evidence to original files, session timestamps/line numbers, Git commits/blobs or historical CI downloads. `MANIFEST.sha256` covers 193 payload files; the two remaining files are the manifest itself and `INTEGRITY.json`.
- **Checks completed:** all 18 original incident files matched their retained source bytes; recorded source hashes and payload checksums passed; the final ZIP passed CRC, member equality, safe-path and permission checks. The Explorer build artifact also matched its GitHub-provided digest.

The archive contains configuration credentials and internal operational details. It is retained with owner-only permissions and is **not a public attachment**. The public excerpts below do not include those credentials. This appendix does not alter the sealed archive: its `original-incident/SRE-POSTMORTEM.md` remains the pre-appendix report captured during collection.

### Evidence Inventory

Paths in this table are relative to the private extracted bundle, not public repository download links.

| Evidence | Retained artifacts | What it supports |
| --- | --- | --- |
| Initial impact and unaffected dependencies | `original-incident/initial-hlf-peer-org.json`, `initial-hlf-explorer.json`, `initial-hlf-ca.json`, `initial-hlf-orderer.json` | Pod state, termination reasons, restart counts, affected peers/databases and surrounding workload status |
| Controlled diagnosis | `original-incident/diagnostic-status.json`, `runtime-ab-test.txt`; underlying command output in `logs/` | Diagnostic OOM termination, inherited descriptor limit and capped/uncapped Erlang startup comparison |
| Recovery actions and Pod state | `original-incident/backoff-reset.json`, `recovered-hlf-peer-org.json`, `recovered-hlf-explorer.json` | Which peers were reset and the replacement workload state |
| CouchDB and ledger checks | `original-incident/couchdb-up.json`, `ledger-verification.json` | Eight authenticated database health checks, channel heights and block-hash comparisons |
| Explorer identity and trust | `original-incident/check-explorer-identity.js`, `explorer-identity-validation.txt`, `retired-explorer-template.json`; related logs | Executed validation procedure, identity/TLS findings and retained superseded configuration |
| Final stability | `original-incident/final-verification.json`, `started-utc.txt`, `verified-utc.txt`; `logs/38-part-03.log` | Final readiness, restart counts, Argo state and HTTP result; timestamps remain subject to the timeline limitations above |
| Historical command output | **55 output chunks from 44 tool calls** in `logs/`; `session/tool-index.json` and `session/commands/` | Application/kernel log excerpts, diagnostics, source changes, recovery commands and publication checks with collection context |
| Session provenance | **100 scoped records** in `session/incident-records.jsonl`; `session/conversation.md` | Original retained tool records and visible incident/publication conversation; unrelated conversation and internal reasoning are excluded |
| Corrective changes | `git/hlf-peer-org-fe1b20d/`, `git/hlf-explorer-3e349bf/`, `git/hlf-peer-org-79be4ee/` | Exact commit records, patches and available before/after file blobs for both fixes and postmortem publication |
| CI and build evidence | `ci/hlf-peer-org-34232370232/`, `ci/hlf-explorer-34233156548/` | Both successful historical runs, job metadata, original/extracted CI logs and the Explorer Docker build record |
| Later context | `context/persistence-report-september-8-excerpt.md` | Incident section from a subsequently maintained report; explicitly not a contemporaneous September 8 log |

Historical CI links: [peer runtime regression run](https://github.com/HLF-Enterprise-Blockchain/hlf-peer-org/actions/runs/34232370232) at `fe1b20d30f20a32b5b4e36c76c00da6a802b4712`, and [Explorer regression/build run](https://github.com/HLF-Enterprise-Blockchain/hlf-explorer/actions/runs/34233156548) at `3e349bfd8f52af8f634a2e75ddf4406db3ad19c3`. The retained Explorer artifact is `HLF-Enterprise-Blockchain~hlf-explorer~IUEFPO.dockerbuild`, artifact ID `10058681639`. These are historical runs; tests were not rerun to manufacture incident evidence.

### Selected Original Log Excerpts

These are selected lines from retained command outputs, not complete Pod-lifetime logs. Filenames and line numbers refer to the private bundle. Terminal color escapes, where present, are omitted for readability. The private Service IP in the peer excerpt is replaced with `<couchdb-service-ip>`; other quoted content is unchanged. The command index supplies the original command and collection timestamp; kernel bracketed times are not wall-clock timestamps.

**Kernel OOM evidence** — `logs/04-part-02.log`, line 236:

```text
[374015.013343] Memory cgroup out of memory: Killed process 834783 (beam.smp) total-vm:3288792kB, anon-rss:521920kB, file-rss:1760kB, shmem-rss:0kB, UID:5984 pgtables:1172kB oom_score_adj:996
```

**Capped Erlang startup** — `logs/11-part-02.log`, line 1; **uncapped comparison** — `logs/11-part-03.log`, line 1, respectively:

```text
port_limit=65536 memory=23791736 schedulers=20
command terminated with exit code 137
```

**Peer dependency failure** — `logs/02-part-02.log`, line 10:

```text
2026-09-08 13:19:52.570 UTC 000c WARN [couchdb] handleRequest -> Attempt 1 of 11 returned error: Get "http://peer0-cbo-couchdb:5984/": dial tcp <couchdb-service-ip>:5984: connect: connection refused. Retrying couchdb request in 125ms
```

**Historical Explorer TLS failure** — `logs/24-part-02.log`, line 2; **replacement wallet refresh** — `logs/32-part-02.log`, line 11, respectively:

```text
legacy TLS error unable to verify the first certificate
Refreshed mounted identity for network treetracker
```

**Final verification command output** — `logs/38-part-03.log`, lines 1 and 7, selected separately from the same capture:

```text
All 17 affected workloads ready; both Argo applications Synced/Healthy. Explorer age seconds: 339 restarts: [0]
Explorer HTTP 200
```

### Completeness and Capture Limits

- The collection contains all 18 files found in the original incident folder and all retained tool records in the scoped recovery/publication conversation. It does not establish that no other retention system has additional evidence.
- Runtime commands used `--tail`, `--since` and filtering. Their retained outputs are excerpts, not full continuous logs from all 17 Pods.
- Three chunks were already truncated at original capture: `logs/14-part-02.log`, `logs/17-part-02.log` and `logs/21-part-02.log`. The missing text was not reconstructed or represented as original output.
- `runtime-ab-test.txt` and `explorer-identity-validation.txt` are original investigator summaries. The timestamped session commands and outputs supply the underlying observations; summaries are not presented as raw application logs.
- Temporary files `/tmp/explorer-fixed.yaml`, `/tmp/explorer-server-validation.txt`, `/tmp/fix-explorer-wallet.py` and `/tmp/write-fabric-postmortem.py` were no longer present during archival collection. Retained commands/output and Git blobs provide substitutes where available, without claiming identical missing-file bytes.
- No live Loki/Prometheus export, PVC/database dump, signing-key backup or new cluster inspection was collected. Exact outage onset, exact downtime and client transaction-loss counts remain unestablished.

`GAPS-AND-LIMITS.json` preserves these collection boundaries alongside the evidence. Historical failed commands, warnings and the unresolved legacy CBO Readers-policy restriction remain part of the record.
