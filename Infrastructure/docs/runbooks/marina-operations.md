# MARINA operational hardening

Infrastructure owns backup orchestration, metrics discovery, alert rules and
dashboards. Lighthouse owns metrics and metadata/original restore semantics;
Fleet owns immutable images, initialization and workload rollouts.

## Status and activation gates

Phase 9 is accepted, including the operator-approved historical backup timing
exception. Phase 10 monitoring is deployed and verified; backup automation is
explicitly deferred, not production accepted.
On 2026-11-03 the operator selected backups only on the Windows cluster host
and no external notification destination. Off-host custody, independent-host
recovery and notification delivery are explicitly deferred, not prerequisites
for this reduced local scope. Local backups cannot protect against loss of
that host/disk, and failures require manual dashboard/journal inspection.
The operator additionally deferred local backup implementation and scheduling
on 2026-11-03 at 22:15 +07:00. Do not enable a timer or implement the local mode
without a new request. Retain the existing encrypted recovery archives.

Infrastructure `d20a517` and Fleet `a62a93a` reconciled the telemetry release.
Both internal API/worker Prometheus targets are UP, all 12 MARINA rules are
healthy/inactive, and dashboard application panels are present. API/worker
readiness passes, public API `/metrics` remains absent, authenticated PDF
source retrieval passes and existing PVCs are retained. External notification
delivery is intentionally unconfigured.

The existing command and systemd templates below still implement the earlier
off-host design and deliberately reject local destinations. Do not enable them
for local backups or bypass their mount guard. An explicit local-backup mode,
local encryption/key-custody configuration and an unattended local restore
verification are still required before activation. Retain the November 3
DPAPI archive as a local recovery checkpoint, not host-loss protection.

## Daily encrypted backups

[marina-backup.py](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/marina-backup.py) is an explicitly gated Linux/WSL
host command. It is not a cluster CronJob and has no automatic deploy hook.
It requires Python 3.10+, `kubectl`, `findmnt` and upstream
[`age`](https://github.com/FiloSottile/age). The scheduler holds only the public
recipient; **never place the recovery private identity or Vault unseal shares
on the Windows/WSL control-plane host**.

1. On the independent recovery host, generate an age identity with
   `age-keygen -o <private-identity-file>`. Protect it in an approved external
   password manager/offline store and test recovery from that store. Copy only
   its `age1...` public recipient to the scheduler. No secret belongs in Git.
2. Provision an authenticated NFS/SMB/SSHFS mount from a physically independent
   host at `/mnt/marina-backups`. Restrict access and pin server identity in the
   mount configuration. The mount must be available to the systemd service
   account, without interactive login. Merely using another WSL distribution
   or a share hosted by the Windows control plane is not independent custody.
3. Copy the [configuration example](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/systemd/marina-backup-config.example.json)
   outside the repository to `/etc/marina-backup/config.json`; replace every
   placeholder. `custody_host` must match the mount source. The program refuses
   local filesystem fallback, loopback hosts, missing mounts and a private key
   in place of the public recipient. An operator must still verify that the
   named host is physically independent; filesystem detection cannot prove it.
4. Create a dedicated `marina-backup` account, private
   `/var/lib/marina-backup`, a mode-0600 kubeconfig at
   `/etc/marina-backup/kubeconfig`, and the reviewed Infrastructure checkout at
   `/opt/homelab/Infrastructure`. Grant only the required control operations:
   get deployments/pods, create/exec/delete backup pods in `marina-core`,
   scale its two writers, and get/patch the `fleet` Kustomization and
   `lighthouse` HelmRelease in `flux-system`. Creating pods with referenced
   credentials is privileged; protect this account like an administrator.
5. Run once in a maintenance window, with **no concurrent release/deployment**:

   ```bash
   python3 /opt/homelab/Infrastructure/scripts/marina-backup.py \
     --config /etc/marina-backup/config.json --acknowledge-maintenance
   ```

6. Verify ciphertext and its receipt on the external host, perform the recovery
   drill below, and configure an operator notification for systemd failures or
   a `last-run.json` older than 26 hours. Journald is not notification delivery.
   Install the reviewed [service](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/systemd/marina-backup.service)
   and [timer](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/systemd/marina-backup.timer) into systemd, then
   explicitly enable `marina-backup.timer`. Inspect `systemctl list-timers`
   and `journalctl -u marina-backup.service` after the first scheduled run.
   WSL must actually be running at the scheduled time; `Persistent=true`
   catches a missed run on restart, but does not keep Windows or WSL awake.

The timer runs daily at 03:00 local time with up to 15 minutes jitter. Backups
retain 30 days of timestamped, verified ciphertext/receipt pairs, always keeping
the newest. Unrecognized files, symlinks, partial files and archives without
matching hash receipts are not pruned. A failed transfer/encryption never counts
as success. Review destination free space; every source archive is capped by a
256 MiB temporary volume, so exceeding that bound fails visibly.

The command takes a nonblocking local lock, refuses suspended/reconciling
releases or unexpected writer/image baselines, prepares a bounded helper with
read-only original PVC access and no service-account token, temporarily pauses
Fleet/Lighthouse reconciliation, stops API admission, drains jobs with the worker
still running, then stops the worker. Ingress has no ready API backend during
maintenance; no backup traffic is public. `marina-admin backup` rechecks that
jobs are drained and hashes every original. Only ciphertext streams to the host
mount; plaintext exists only in the helper's ephemeral storage and pipe.
Service/reconciliation restoration and helper removal run on errors and SIGTERM.
SIGKILL, host/power loss or Kubernetes loss cannot execute cleanup: immediately
check writer replica counts and suspensions before manually restoring their
recorded baseline. No PVC is removed or replaced.

Receipts identify the immutable image, ciphertext hash, retention and custody
reference. A backup success is **not** a restore success. A service-restoration
failure exits nonzero and is recorded in local `last-run.json`, even if the
ciphertext itself was completed.

## Independent recovery and rehearsal cadence

- **Weekly:** decrypt the newest retained copy using the identity recovered
  from external custody, not a cached identity on the scheduler.
- **Monthly and before schema/storage upgrades:** perform the full isolated
  restore/reindex drill on an independent host and retain its report beside
  the encrypted backup. Keep at least the last successful rehearsal report.
- **Every release:** rehearse upgrade and rollback against disposable targets.
  Record both digests, schema versions, readiness, exact citations and hashes;
  never use the live database as a rollback experiment.

On the independent recovery host, verify the encrypted receipt hash first.
Decrypt into private temporary storage on encrypted disk or RAM, never the
repository or a shared plaintext directory:

```bash
umask 077
age --decrypt --identity <externally-recovered-identity> \
  --output <private-temporary-snapshot.zip> <retained-snapshot.zip.age>
```

Provision a **new empty PostgreSQL database** named `marina_drill_<32 hex>`,
an empty `<same-name>/originals` directory (parent must exist), and a previously
unused Qdrant collection name identical to the database name. Set isolated
`MARINA_DATABASE_URL`, filesystem storage path, vector collection and the exact
archived embedding model/digest settings in a private environment, never by
editing a production Secret. The [restore drill](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/marina-restore-drill.py)
runs in the pinned Lighthouse image/environment with its PostgreSQL client tools:

```bash
python marina-restore-drill.py --archive <private-temporary-snapshot.zip> \
  --report <new-report.json> --acknowledge-isolated-target
```

It refuses production-shaped database/storage/vector names, nonempty restore
targets and existing reports. It restores metadata/originals, migrates, reindexes
through real worker/model/vector dependencies and compares exact semantic version
metadata, chunk IDs/text/lines/pages, original bytes and active vector counts.
Failed/deleted records are preserved; they are not silently made Ready.
Targets are deliberately retained for inspection on success or failure.
After reviewing the report, remove **only** the explicitly named disposable
database, vector collection, originals and temporary plaintext archive. Never
redirect production traffic to a partial restore.

An image rollback does not reverse database migrations, PVC writes or storage
initialization. In particular, pre-Phase-9 Lighthouse readiness requires `0001`
and fails against `0002`. Either promote a reviewed compatibility image or
restore the retained pre-upgrade backup into fresh `0001` targets before using
the older image. Preserve live volumes until recovery is accepted. See the
[application rollback procedure](../../../lighthouse/PRODUCTION-RUNBOOK.md#6-rollback).

Use the [schema probe](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/marina-schema-probe.py) inside each pinned
image with the isolated runtime environment to record `/ready` without applying
migrations:

```bash
python marina-schema-probe.py --expected-status 200
```

For an intentionally incompatible previous image on the upgraded schema,
`--expected-status 503` requires that **only** the database check fails, rather
than mistaking a model/storage outage for proof of schema incompatibility.

### November 3 isolated upgrade/rollback record

Using disposable PostgreSQL 17.6 databases, originals and Qdrant collections:

- Previous image `sha256:19c228e77cb6936a756deff8129d19105ff9ddf4ed6495bd41ea7f14a031e271`
  was ready on restored `0001`.
- Current Phase 9 image
  `sha256:e56901495ff289aabedeb975840e8c81bc8527280c3573d6770579020a02e726`
  upgraded that target to `0002` and was ready with all checks true.
- The previous image on `0002` returned 503 with only `database=false`,
  confirming that an image-only rollback is not accepted.
- Restoring the pre-upgrade synthetic backup into another fresh `0001`
  target allowed the previous image to reindex with identical text citation
  IDs/content and return 200 with all readiness checks true.
- The new restore tool independently restored the synthetic text/PDF `0002`
  archive: 2 originals and 4 exact page-aware chunks/vectors, with both documents
  reindexed. This verifies the tool against real PostgreSQL, model and Qdrant
  dependencies, not an independent-host custody claim or a live Helm rollback.

The test targets were removed after reports were retained. No production
database, originals PVC, workload image or release setting changed in this
rehearsal. Repeat with the next candidate images before promotion.

Vault custody is separate: take an authenticated Raft snapshot under the
[Vault procedure](https://github.com/nnvuf-homelab/infrastructure/blob/main/docs/runbooks/vault.md#backup-and-host-loss-precautions), retain the TLS material
and distinct existing unseal shares in separately controlled external stores,
and verify recovery on an independent host. Do not copy shares into backup
configuration, provision a root token for automation or claim that a Lighthouse
archive backs up Vault. The actual host, recovery custodians, off-host copy and
independent decryption/restore evidence remain operator gates.

## Metrics and rollout order

Promote the telemetry-capable Lighthouse image through Shipyard/Fleet first.
Then explicitly provision runtime configuration with
`setup-lighthouse.sh --provision-runtime --context <approved-context>` and roll
API/worker through the reviewed deployment flow so they inherit
`MARINA_METRICS_PORT=9090`. ConfigMap changes alone do not restart existing pods.
Finally reconcile the Infrastructure metrics Services/ServiceMonitor and rules.
Expect `MarinaMetricsUnavailable` until this handoff is complete.

Only ClusterIP Services expose port 9090; no ingress/tunnel route points at it.
The public 8080 API does not expose `/metrics`. Counters reset on process
restart; dashboards use `rate`/`increase`. No document/owner/query labels are
exported. Queue gauges come only from the API to avoid duplicate counts.
Missing or failed scrapes are not interpreted as zero failures.

## Alert responses

All 12 committed MARINA rules have deterministic `promtool` firing, hold-time
and recovery coverage in [the rule tests](https://github.com/nnvuf-homelab/infrastructure/blob/main/tests/marina-alerts.test.yaml).
This proves rule evaluation, not that Alertmanager reaches an operator.
Configure and test an approved receiver before Phase 10 acceptance.

### MarinaQueueStalled
Oldest uploading/queued/running job >300 seconds for 5 minutes. Inspect worker
readiness, job state and model/vector dependencies. Recover the dependency
before owner-scoped retry; do not delete queue rows or blindly restart writers.

### MarinaFailedJobs
Terminal failed jobs persist for 5 minutes. Read the actionable document error;
unsupported PDF failures need repaired/OCR text-layer input, not an automatic
retry loop. Retry only after correcting the cause, or deliberately delete.

### MarinaDependencyUnavailable
A readiness dependency is zero for 3 minutes. Check the named dependency and
API/worker `/ready`; use the [outage triage runbook](https://github.com/nnvuf-homelab/infrastructure/blob/main/docs/runbooks/outage-after-restart.md).
Do not replace a PVC or weaken authentication to restore readiness.

### MarinaMetricsUnavailable
A metrics target is down/missing for 5 minutes. Check image/config rollout,
9090 listener, Service selectors and ServiceMonitor discovery. A failed
database scrape must stay visibly failed rather than supplying healthy zeros.

### Availability and resources

| Alert | Response |
| --- | --- |
| MarinaDeploymentUnavailable | Check desired/ready replicas, rollout events and `/ready`; preserve inactive zero-replica roles. |
| MarinaStatefulSetUnavailable | Check PostgreSQL events, node and retained claim; never recreate a bound claim to clear the alert. |
| MarinaServiceHasNoReadyEndpoints | Trace Service selectors, EndpointSlices and pod readiness before changing ingress. |
| MarinaContainerRestarting | Inspect previous container logs and OOM/exit events; correct the cause before restarting. |
| MarinaContainerMemoryNearLimit | Check working set, parser/worker concurrency and OOM events; review limits and node headroom. |
| MarinaContainerCPUThrottled | Check sustained load and request latency before reviewing CPU limits. |
| MarinaPersistentVolumeFilling | >80% for 15 minutes: verify backup and actual volume stats, inspect originals/PostgreSQL growth; do not delete data or assume local-path resize. |
| MarinaEdgeHigh5xxRate | Trace Cloudflare, Traefik, Service readiness and backend logs; a 302 Authentik challenge is not a 5xx. |

## Harbormaster

Read-only production MCP runs as `mcp-gateway` in `marina-mcp`; Fleet owns its
image/configuration. Check `/healthz`, `/ready`, protected-resource discovery,
then a real user's list/search/source call. Discovery alone does not prove
authorization. For 401/403, check time/issuer/audience/scopes and
`marina-mcp-users` membership under the [OAuth runbook](marina-mcp-oauth.md).
Never add a shared server credential or broaden the three-route allow-list.
Do not log queries, cookies or access/refresh tokens. Roll back only to a
compatible image preserving per-request identity and current OAuth settings.
Two-user isolation and client silent renewal remain separately tracked gates.

## Trawler

`fetch-worker` in `marina-fetch` is a private manual operator helper; it has no
public ingress or automatic consumers. Keep one replica with Recreate and the
retained SQLite PVC. Check `/ready`, acquisition outcome/error and journal
integrity; retry retained acquisition handoff with the user's current session
without giving the service Lighthouse credentials. Private/LAN URL refusal is
intentional; do not bypass target validation to repair a failed fetch.
The journal retains normalized text/provenance, not a complete HTML/image
archive. Follow the [storage runbook](trawler-storage.md) for SQLite online
backup and controlled restore. The Lighthouse backup command does **not** cover
this journal; its separate off-host backup/custody remains an activation gate.
