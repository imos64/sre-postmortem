# Operations, recovery verification and rollback

[Incident guide](README.md) · [Full postmortem](2026-09-19-windows-wsl-runner-recovery-postmortem.md) · [Actions](ACTIONS.md)

This is the recorded procedure for the repaired NucBoxK10 installation. The final
qualification is complete; no reboot, failure injection, sign-in-policy change,
or rollback is pending. Recovery scripts and private original backups remain on
the host. This public documentation is not a portable installer.

## Check the running platform

From Windows PowerShell:

```powershell
wsl -d Ubuntu-24.04 -u imos -- /usr/local/bin/imos-platform-health
```

For structured details on the same running platform:

```powershell
wsl -d Ubuntu-24.04 -u imos -- /usr/local/bin/imos-platform-health --json
```

The Linux executable is read-only and does not poll for jobs or start services.
The surrounding `wsl.exe` command can launch a stopped distribution, which can
start enabled Linux services. During a future reboot qualification, preserve and
read the automatically recorded evidence first; otherwise this command could
become a startup stimulus and invalidate a claim of automatic recovery.

A successful exit requires live Docker and enrolled cluster/node identities,
ready workloads and storage checks, active connector and real runner, fresh
accepted heartbeat evidence, current Windows/WSL identity and held keeper lock,
fresh supervisor observations, and current-boot full verification. The heartbeat
must match the running process and boot and be accepted within 45 seconds; the
supervisor observation must be current within 120 seconds. A process name or
`systemctl is-active` alone is insufficient.

| Output | Operational interpretation |
| --- | --- |
| `READY` | Required health checks passed without recorded warnings |
| `READY_WITH_WARNINGS` | Required checks passed; inspect the explicit warning list |
| Nonzero exit / failed checks | Preserve the failed output, identify the failed layer and inspect current evidence before changing services |
| Runner `READY` with recent accepted heartbeat | Real runner authentication and accepted control-plane communication are current within the check's limits |

The final snapshot contained two existing warnings:
`kind-dev:no-ready-endpoint:hlf-orderer/raft-orderer-headless` and
`kind-dev:no-ready-endpoint:hlf-orderer/v2-raft-orderer-headless`.
They require a separate owner review of intended consumers. Do not suppress them
or alter the dashboard Offline predicate to manufacture a passing status.

## Evidence locations and diagnosis order

The private incident root is
`/home/imos/incidents/2026-09-19-runner-recovery/`. Its `RESUME.md` marks the repair
completed, and `FINAL-VERIFICATION.json` joins Linux, Windows, independent remote
and owner dashboard evidence. The raw Linux result deliberately defers sign-in
qualification to the separate Windows timeline.

| Layer | Evidence to inspect privately |
| --- | --- |
| Windows boot launch | Existing `TreeTracker kind-dev startup` task Action/trigger, runtime and logs under `%LOCALAPPDATA%\ImosPlatformRecovery\logs` |
| Sign-in observer | `Imos-Control-Planes-77ff47b974`; `~/.local/share/imos-autostart/` configuration and log |
| WSL lifetime | Foreground keeper lock and current Windows/WSL marker; do not rely on kernel boot ID alone |
| Root recovery | `imos-platform-recovery.service` journal and `/var/lib/imos-platform-startup/supervisor.json` |
| Full readiness | `/var/lib/imos-platform-startup/latest.json` and its referenced evidence directory |
| User services | Effective unit definitions and journals for `imos-benchmark-runner.service` and `imos-kube-connector.service` |
| Runner acceptance | `~/.local/share/imos-benchmark-runner/recovery/status.json`; current-boot/PID freshness plus independent accepted remote last-seen |
| Job safety | Existing local active/report records, child processes and safe remote queue state; never print leases, keys, task content or raw reports |

Inspect the effective next-start command, not only the base unit file. If a
service has been active since before a configuration change, its current process
may conceal a different next-start command. Preserve failure receipts before
reloading units or starting components.

Recovery automatically retries delayed dependencies. Identity mismatches require
review rather than bypassing guards. Do not recreate clusters, prune Docker, reset
pairing, broaden firewall rules, delete a runner lock, or replay saved jobs as a
routine recovery step. Pause disruptive testing while local or remote work is active.

## Future reboot qualification

This procedure is retained for a future deliberately scheduled test, not a new
reboot request. The one-shot proof recorder is currently disabled and its arm is
archived. A future test requires a fresh arm tied to the current boot and fresh
idle checks; the old archived acceptance cannot qualify another boot.

1. Save Windows work and verify local children, active/report state, and remote
   queues are idle. Postpone on dependency errors, uncertain job state or active
   work. Preserve current identities, status and configuration receipts.
2. Save a persistent handoff, arm the reviewed observer for a different Windows/WSL
   boot, and verify its output location. Do not change the user's sign-in policy.
3. For the before-sign-in condition, completely sign out, then restart from the
   Windows sign-in screen. Remain signed out for at least five minutes after the
   final sign-in screen appears, and long enough for the automatic observer to
   finish. Do not open Ubuntu or run WSL/startup commands during observation.
4. After returning, read existing Windows task/session logs and the automatic
   Linux proof before issuing a live health command. Compare Windows boot and
   sign-in events with wrapper/keeper start, full readiness and accepted polls.
5. Require full readiness and multiple accepted heartbeat timestamps before
   interactive sign-in, one valid keeper, current identities, and no investigator
   startup commands. An early sign-in produces an incomplete test, even if the
   platform later becomes Ready.
6. Then confirm independent advancing control-plane acceptance across multiple
   polling intervals, owner dashboard Ready, and scoped file/job/storage parity.
7. Archive both failures and successes. Disable only the completed test recorder;
   leave normal recovery supervision and the foreground keeper intact.

The completed final test observed 12 accepted timestamps over 180.21 seconds and
later independent remote samples over 126.439 seconds. Those are historical
observed windows, not a substitute for collecting fresh future evidence.

## Controlled dependency and idle process-failure tests

The incident tested a runner-only Docker shim and a verified-idle runner SIGKILL.
It did not stop the shared engine, disconnect the actual network, or crash an
active benchmark. Before repeating, review the preserved test implementation,
check the current runner interface and workloads, and prepare fixture removal.
Only genuine accepted responses after failure count as recovery. Unit mocks or
injected success receipts cannot satisfy service-level acceptance.

A shared-engine or sustained real-network outage test would have broader impact
on the 131 enrolled controllers and needs its own maintenance plan. The current
published evidence must not be described as having performed that test.

## Guarded rollback

Rollback was prepared and both dry-runs passed; it was **not applied**. Restoring
the original standby policy will make the dashboard Offline on the next runner
start. Check local and remote work is idle and use the exact private receipts;
do not force a rollback after later file/task changes invalidate the guard.

1. From an elevated Windows PowerShell, execute the preserved Action rollback:

   ```powershell
   powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -File 'C:\Users\Imos\AppData\Local\ImosPlatformRecovery\backups\20260919T220808706Z\rollback-supervised-task.ps1' -Apply
   ```

   Omit `-Apply` for its dry-run. It restores the saved task Action only after
   checking current task/script state. It does not stop the current task or WSL
   keeper. This uses a per-process execution option, not a machine policy change.

2. From the existing Ubuntu session, run the Linux dry-run:

   ```bash
   /mnt/c/windows/system32/wsl.exe -d Ubuntu-24.04 -u root -- /usr/bin/python3 -I /home/imos/incidents/2026-09-19-runner-recovery/rollback-linux.py
   ```

3. Append `--apply` only for an intentional rollback after the dry-run passes.
   The dry-run verifies installed-file and backup hashes. The apply guard also
   checks the local idle receipt, absence of active/report records, and the
   coordinator PID. It does not independently query remote queues or prove
   receipt freshness: repeat the operator's fresh local/remote idle check
   immediately before applying. It restores 15 guarded files and
   modes, reloads systemd, and stops/disables only the new recovery/proof units.
   It does not stop cluster containers, the foreground keeper or benchmark jobs.
4. The current runner retains its loaded code. When idle, a deliberate
   `systemctl --user restart imos-benchmark-runner.service` activates the restored
   standby; alternatively the next normal reboot activates it. This is expected
   rollback degradation, not restored online operation.

Private receipts are `linux-installed.json`, `runner/installed-files.json`,
`observer-installed.json`, the Windows Action receipt and original backup
folders. The completed proof recorder is already disabled. Earlier tunnel/API
address repairs are separate and remain installed; their coordinated rollback
must follow their own identity/resource-version guards. Restoring their old
hardcoded addresses would reintroduce the earlier ingress incident.
