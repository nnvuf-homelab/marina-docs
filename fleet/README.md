# Fleet

This repository is the application onboarding fleet for the homelab cluster.

- `Infrastructure` owns system infrastructure and Flux bootstrap.
- `shipyard` owns application CI/CD eligibility and image promotion.
- This repository owns which application repositories are admitted to Flux.

Each application has a directory under `apps/` with a Kustomize entry point.
Application source and CI remain in their application repositories; this catalog
owns deployment manifests and Flux admission. Shipyard promotion pull requests
generate Helm charts and releases pinned to the published image digest.

## Layout

```text
fleet/
  apps/
    lighthouse/          # Same Helm layout for chartroom, trawler, harbormaster
      chart/
        Chart.yaml
        values.yaml
        templates/
      release.yaml
      kustomization.yaml
    minimal-service/
      source.yaml
      kustomization.yaml
      deploy/            # Active legacy Kustomize deployment
  kustomization.yaml
```

The fleet is reconciled by Infrastructure through a single Flux source and
Kustomization at `./`, using the root `kustomization.yaml` as the admission
allow-list. This private repository requires a Flux Git Secret created
locally; the credential is never committed here.

## Admitted Applications

The `apps/lighthouse`, `apps/trawler`, `apps/chartroom`, and
`apps/harbormaster` deployment paths are admitted in the root allow-list.
Each application entry point includes only its `release.yaml`; Helm owns the
workloads rendered from `chart/`. Git-backed charts use
`reconcileStrategy: Revision` so image digest changes trigger chart updates.

The superseded MARINA `deploy/` scaffolds have been removed. Do not recreate or
apply them alongside Helm releases. The `apps/minimal-service/deploy` directory
is still referenced by its Flux Kustomization and must remain.

Admission does not prove workload readiness. Check HelmRelease status and pod
readiness after promotion; private GHCR images require namespace-local registry
credentials, and public ingresses require the platform middleware and TLS
secrets.

## MARINA production safety hold

Lighthouse and Chartroom releases are suspended as a production safety hold
until the [production gates](../lighthouse/PRODUCTION-RUNBOOK.md) are verified.
Lighthouse's release references Infrastructure's `lighthouse-runtime` ConfigMap
and `lighthouse-credentials` Secret in `marina-core`; missing inputs fail
closed. Infrastructure's configuration-only provisioning command writes the
ConfigMap; External Secrets builds the application Secret from Vault's existing
Lighthouse application database password. Verify `vault-lighthouse-postgres`
and `vault-lighthouse-credentials` are Ready before resuming the release.
Both public routes retain HTTPS redirect and Authentik forward-auth.
Chartroom uses Authentik's browser session, and Lighthouse revalidates the
Cookie against the configured internal outpost instead of trusting identity
headers. Production needs no dedicated OAuth client or manually entered token.
Only API and worker run initially; RAG and learning
replicas are intentionally zero. Shipyard regeneration preserves these
Fleet-owned release settings. Do not edit generated chart defaults to persist
runtime overrides.

### Private filesystem originals

Infrastructure owns `lighthouse-originals`, a retained 20Gi `local-path`
ReadWriteOnce PVC. The runtime selects `filesystem` at
`/var/lib/marina/originals`. Initialization and application pods mount the same
claim at `/var/lib/marina` with `fsGroup: 65532`; the non-root app creates its
private originals subdirectory. No S3 credentials or public file-serving route
is required.

Fleet's Deployment post-renderer sets mounts, group permissions, and `Recreate`
strategy. It is preserved by the currently pinned Shipyard generator even when
that generator refreshes the Deployment template without volume support.
The initialization hook template mounts the claim directly: Helm hooks are not
patched by Flux post-renderers. Keep both paths in sync.

ReadWriteOnce permits several pods on the same node. The PVC's node affinity
co-locates API, worker, and initialization; it does not provide multi-node
failover or replicas of the files. Preserve off-node backups and plan a storage
migration before enabling cross-node scaling. `Recreate` can cause brief
per-role downtime on upgrades. Never delete the claim during code rollback.
Existing S3 originals require an explicit quiesced migration/verified restore,
not a runtime switch to an empty directory.

### Release-controlled initialization

Lighthouse enables `initialization` in HelmRelease values. The chart's
`templates/initialize-job.yaml` renders a `lighthouse-initialize` Job as a
Helm `pre-install,pre-upgrade` hook (no post hooks), so workloads are not
created or updated until it succeeds. The Job uses the release's exact
`image.repository@image.digest`, the same `runtime.envFrom` references,
`ghcr-pull`, non-root/seccomp/dropped-capability security, soft workload
affinity, and the release resources. Its command refuses to run unless
`MARINA_AUTH_MODE=forwardauth` with valid fixed outpost URL, public host,
browser Origin, and stable owner namespace, then runs
`python -m marina.initialize` (migrations plus storage initialization).

- `backoffLimit: 0` and `activeDeadlineSeconds: 600` bound each attempt; the
  release `timeout: 15m` covers image pull, the hook deadline, and workload
  readiness. A failed or timed-out Job fails the install/upgrade.
- `hook-delete-policy: before-hook-creation,hook-succeeded` removes successful
  Jobs and keeps the latest failed Job for inspection until the next attempt
  replaces it. Initialization is idempotent, so Flux remediation retries are
  safe.
- Migrations and storage initialization are **not** undone by Helm rollback or
  Flux upgrade remediation (`strategy: rollback`). Rollback restores the previous
  workload manifests only; the previous image must tolerate the migrated schema,
  otherwise restore data from backup per the runbook.
- Generated charts for other applications ship the same template disabled
  (`initialization.enabled: false`); it renders nothing unless Fleet enables it.
  Older Shipyard generators rewrite only `deployment.yaml`, `service.yaml`,
  `ingress.yaml`, chart values, and the release spec they merge, so this
  template and the release `initialization`/`timeout` values survive
  regeneration. YAML comments in `release.yaml` are not preserved.

Suspension takes effect after reconciliation and does not terminate already
running workloads. Resume only through a separately approved change after the
runbook prerequisites pass. Never put credentials in release values.

MARINA charts reference `ghcr-pull` in their target namespaces. Verify Vault's
External Secrets synchronization is Ready before reconciliation, including
`ghcr-pull` and `cloudflare-origin-tls` in `marina-core` and `marina-ui`.
Infrastructure's `scripts/setup-secrets.sh` is bootstrap/break-glass only.
The chart `imagePullSecrets` list can be overridden in HelmRelease values;
minimal-service retains its separate `ghcr-minimal-service-pull` reference.
