# Trawler acquisition storage

## Ownership and lifecycle

- Infrastructure owns the retained `marina-fetch` namespace and
  `trawler-acquisitions` PVC through
  [marina-storage](https://github.com/nnvuf-homelab/infrastructure/blob/main/cluster/applications/marina-storage/trawler.yaml).
  Both objects disable Flux pruning. This does not prevent manual deletion.
- Fleet owns the workload mount and rollout settings in
  [the Trawler HelmRelease](https://github.com/nnvuf-homelab/fleet/blob/main/apps/trawler/release.yaml).
- The single replica mounts `/var/lib/trawler`; the journal is
  `acquisitions.sqlite3`. Pod `fsGroup: 65532` permits non-root writes.
  `Recreate` avoids overlapping writers during upgrades, at the cost of brief
  downtime. Do not scale this SQLite service above one replica.
- Storage is `2Gi`, `ReadWriteOnce`, `local-path`, bound to `windows-master`.
  The request is not a hard filesystem quota. There is no automatic journal
  retention or backup schedule yet; monitor host disk space and journal growth.
  Do not assume online expansion is supported by this StorageClass.
- There is no cross-node failover. Moving to another node requires a verified
  backup, a separately provisioned destination, and a controlled restore.

The acquisition API remains private, without an Ingress, service-account token
or backing-store credentials. NetworkPolicy manifests alone do not provide
isolation on this cluster's Flannel configuration.

## Accepted migration - 2026-11-03

Infrastructure `c2faf8a` provisioned the PVC. Fleet `1ed6e58` added the mount;
`cad6dce` explicitly cleared the obsolete rolling-update configuration.
The transition initially failed Kubernetes admission because server-side apply
retained `strategy.rollingUpdate` while setting `type: Recreate`. With Helm
reconciliation suspended, a one-time merge patch removed `rollingUpdate`;
resuming and resetting reconciliation then succeeded. No PVC replacement was
needed.

Before switching storage, traffic was paused, in-flight work drained, and a
SQLite online backup was taken. The original pod was stopped before a temporary
non-root restore pod populated the PVC. The helper was subsequently deleted.

Verified evidence:

- The restored backup matched SHA-256
  `6a32f0ea6eda2518b1d21a4e0e4542061b98dde9f48d8f459f1ff01d3ea0897f`.
- Both original records survived, including the recorded failed acquisition.
- A new `https://example.com/` acquisition returned 201; a loopback target
  returned 422 and retained an explicit error.
- All four complete database rows survived replacement of the mounted pod
  unchanged. `PRAGMA quick_check` returned `ok`; the PVC UID stayed
  `72ee007f-762c-455f-97f4-f9006711587e`.
- HelmRelease reconciliation is Ready and unsuspended at
  `0.1.0+cad6dcef018c`. The normal Service selector was restored; readiness
  through the ClusterIP Service returned 200.
- The promoted image remains
  `ghcr.io/nnvuf-homelab/trawler@sha256:98f6fa6fa46f002f772282975535f9d1980607369254bfe55e6ea9edd091ea7b`.

The workstation's local session recovery artifacts are
`trawler-pre-pvc.sqlite3`, `trawler-pvc-migration.json` and
`trawler-pvc-acceptance.json`. Keep the backup private and retain it until a
newer verified backup supersedes it. A local copy is not off-host recovery.
These artifacts are not stored in Git.

Routine readiness, retry and alert responses are summarized in
[MARINA operations](marina-operations.md#trawler). The Lighthouse backup timer
does not include Trawler's journal; retain its separate SQLite backup until
independent journal custody is verified.

## Backup, restore and rollback

1. Use SQLite's online backup API for a consistent snapshot; do not copy a live
   database file directly. Validate the snapshot with `PRAGMA quick_check`,
   record its hash and row count, and retain it outside the pod.
2. Before restoring, suspend the HelmRelease, pause new Service traffic, allow
   in-flight acquisitions to drain, and stop the sole writer. Keep the existing
   PVC intact. Back up the current journal before replacing any data.
3. Use a temporary, non-root, no-service-account-token helper mounting the same
   claim. Restore the verified snapshot and validate integrity and records
   before restarting the workload. Remove the helper, resume reconciliation,
   verify readiness and restore the normal Service selector.
4. Roll back application images only while preserving the PVC, mount and
   single-writer configuration. Reverting to the old ephemeral template would
   hide the retained history and lose new container-local records on restart.
   Never delete the bound claim or namespace to repair a rollout.

## Subsequent controlled-ingestion acceptance - 2026-11-03

Fleet PR #22 merged as `7451129`, deploying the validated-address fallback fix
from Trawler `2bea1f7`. The image is now
`ghcr.io/nnvuf-homelab/trawler@sha256:13f6b3e77a682b241eb1f0067a9d2c3a63f1eb9d8bd46dae9dcab8fa54871e1f`.
Helm reconciliation passed with the same PVC UID, mount, group and Recreate
strategy. Historical records and SQLite integrity remained intact.

The official Foundry guide fetched successfully through the promoted API.
The operator uploaded retained acquisition
`e53a08ff-1f84-4606-a5c3-672bc8c8058f` using their private session; Lighthouse
document `34e762e2-6c9e-4d47-aeb0-18bc10c1ebe3` became Ready/version 1.
Repeated handoff returned the same document, not a duplicate.

A deliberately invalid session caused a non-followed HTTP 302. The journal
recorded the failed handoff without losing fetched content; a valid-session
retry cleared the error and returned the same indexed document. MARINA search
and source retrieval verified its SDK installation and login passage.
Chartroom source inspection passed. Answer completeness and numbered-step
presentation remain separate Lighthouse follow-up work, not storage failures.
