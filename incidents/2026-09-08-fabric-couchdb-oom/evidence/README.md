# Public evidence

These records support the [postmortem](../2026-09-08-hyperledger-fabric-outage-postmortem.md). They describe the
September 8, 2026 development incident and recovery; they are not current health checks.

| File | What it records |
| --- | --- |
| [kernel-oom.txt](kernel-oom.txt) | Kernel OOM kill of CouchDB's Erlang process |
| [erlang-capped.txt](erlang-capped.txt) / [erlang-uncapped.txt](erlang-uncapped.txt) | Successful capped startup and uncapped exit 137 |
| [runtime-ab-test.txt](runtime-ab-test.txt) | Original investigator summary of the controlled comparison |
| [peer-couchdb-failure.txt](peer-couchdb-failure.txt) | Peer connection refusal while CouchDB was unavailable |
| [explorer-tls-failure.txt](explorer-tls-failure.txt) | Historical orderer trust failure |
| [explorer-wallet-refresh.txt](explorer-wallet-refresh.txt) | Replacement wallet initialization refreshed the mounted identity |
| [explorer-identity-validation.txt](explorer-identity-validation.txt) | Original investigator summary of identity/TLS checks |
| [couchdb-up.json](couchdb-up.json) | Eight successful authenticated database health responses |
| [ledger-verification.json](ledger-verification.json) | Channel heights/hashes, channel-less peers and retained legacy policy failures |
| [final-verification.json](final-verification.json) | All 17 workloads ready and both Argo applications Synced/Healthy |
| [final-verification-output.txt](final-verification-output.txt) | Final stability summary and HTTP 200 result |
| [ci-runs.json](ci-runs.json) | Historical CI outcomes at the corrective commits |
| [PROVENANCE.json](PROVENANCE.json) | Source records, selected line numbers, transformations and hashes |
| [SHA256SUMS](SHA256SUMS) | Checksums for the public evidence files |

The five original summary/verification files are copied byte for byte. The seven
command-output files are selected excerpts; terminal color escapes are removed
and the peer excerpt's private Service IP is replaced with `<couchdb-service-ip>`.
Artifact type and transformations are recorded per file in `PROVENANCE.json`.
No missing log text has been recreated.

The complete 195-file archive remains private because it contains configuration
credentials. Full session records, raw configuration snapshots, CI log archives
and build records are not included here. Upstream Git and CI links require access
to the private source repositories. This publication's checksums verify the
published bytes; they do not independently prove a live incident occurred.

The original captures were bounded by `--tail`, `--since` and filters. Three
private output chunks were already truncated at capture, as documented in the
postmortem. Exact outage onset, exact downtime and lost client-request counts
remain unestablished. No live Loki/Prometheus export or cluster access was
performed during this publication.

From this directory, verify the public files with:

```bash
sha256sum -c SHA256SUMS
```
