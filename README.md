# MARINA documentation

This repository is the canonical home for MARINA product, development,
evaluation, deployment, and operations documentation. Application source,
deployment configuration, and cluster runtime remain in their owning
repositories.

## Products

- [Lighthouse](lighthouse/README.md) — backend architecture and development
  guide
- [Chartroom](chartroom/README.md) — document-library UI and browser operations
- [Trawler](trawler/README.md) — controlled web acquisition
- [Harbormaster](harbormaster/README.md) — read-only MCP gateway
- [Fleet](fleet/README.md) — application admission and release operations

## Lighthouse plans and evaluation

- [Development plan](lighthouse/DEVELOPMENT-PLAN.md)
- [Production runbook](lighthouse/PRODUCTION-RUNBOOK.md)
- [Expansion plan](lighthouse/EXPANSION-PLAN.md)
- [Real-model evaluation](lighthouse/evaluation/README.md)
- [Evaluation corpus](lighthouse/evaluation/corpus.md)

## Operations and architecture

- [MARINA operations](Infrastructure/docs/runbooks/marina-operations.md)
- [MCP OAuth setup and acceptance](Infrastructure/docs/runbooks/marina-mcp-oauth.md)
- [Trawler storage and recovery](Infrastructure/docs/runbooks/trawler-storage.md)
- [Original agentic-knowledge architecture proposal](Infrastructure/docs/analysis/agentic-knowledge-system.md)

The architecture proposal is historical; current behavior and rollout status
are described in the product guides and operations runbooks.

Application source remains in the [Lighthouse](https://github.com/nnvuf-homelab/lighthouse),
[Chartroom](https://github.com/nnvuf-homelab/chartroom),
[Trawler](https://github.com/nnvuf-homelab/trawler),
[Harbormaster](https://github.com/nnvuf-homelab/harbormaster), and
[Fleet](https://github.com/nnvuf-homelab/fleet) repositories. Cluster runtime
configuration remains in [Infrastructure](https://github.com/nnvuf-homelab/infrastructure).
