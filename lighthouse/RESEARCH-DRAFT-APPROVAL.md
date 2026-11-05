# Bounded Research with Draft Approval

**Status: in progress - manual-source vertical slice implemented locally; not deployed.**

This feature extends the [Lighthouse development plan](DEVELOPMENT-PLAN.md).

## Use case and scope

The agreed use case is: "Tell me about Azure Foundry." MARINA first searches
the owner's saved knowledge. If that evidence cannot support an answer and the
user has enabled research for this request, MARINA discovers relevant public
sources, acquires and indexes bounded evidence, and saves a cited summary as a
draft. Clicking **Approve** makes that exact report version ready for citation.
Research permission is not publication approval.

This is interactive research, not `knowledge-learning`. It is a concrete,
bounded exception to the deferred research scope, not authorization for
unrestricted crawling, model training, or live cluster changes. Follow the
[research and workload boundaries](../Infrastructure/docs/analysis/agentic-knowledge-system.md)
without treating the full target architecture as already implemented.

## Current foundations and gaps

- Lighthouse already provides owner-scoped collections, documents, chunks,
  vector retrieval, exact source citations, and durable ingestion/deletion jobs.
- Trawler already provides controlled single-URL public HTTPS acquisition,
  provenance, retained outcomes, retries, and an operator-authenticated upload
  helper. See [Trawler's contract](../trawler/README.md).
- Automatic search discovery is missing. Durable research runs, versioned reports,
  publication approval, and draft-aware retrieval are implemented in the local
  manual-source backend slice described below; live acceptance is still pending.
- The current source-first answer path returns selected source excerpts, not
  newly authored research summaries. Add a separately evaluated research-summary
  path; preserve the accepted source-first question behavior.
- An ordinary document upload is not a staging mechanism: once ingestion finishes,
  that document becomes retrievable. Research requires an explicit visibility
  boundary before reusing ingestion.

## Workflow and approval rules

1. Search authorized saved knowledge and assess evidence sufficiency. A dependency
   outage is an error, not evidence that the topic is unknown. If research is
   disabled, preserve the current insufficient-evidence response.
2. With research enabled and a destination collection selected, create a durable,
   owner-scoped research run. Return its ID promptly so the UI can poll progress
   and cancel without holding the question request open.
3. Discover candidate URLs through a configured search provider, initially
   preferring official documentation for product questions. Enforce explicit
   query, URL, byte, time, and model-call budgets; no recursive crawling or
   unbounded planner loops.
4. Fetch selected URLs through Trawler's existing policy boundary. Retain the
   requested/final URLs, acquisition time, content hashes, and exact normalized
   evidence used by the report. Treat source instructions as untrusted data.
5. Index acquired evidence in research staging, isolated from normal collection
   search and question answering. Research retrieval and owner-authorized draft
   preview may access it; ordinary source/original endpoints must not bypass
   staging restrictions.
6. Generate and save an immutable draft report version with material claims
   linked to exact evidence versions/chunks, model identity, and uncertainty.
   Partial research must expose gaps; an unsupported report cannot be approved.
7. Let the owner inspect the saved draft and its evidence. Approval names the
   expected version and content hash and records the approver and timestamp.
   Reject stale approval requests with a conflict; never approve whatever version
   happens to be latest when the request arrives.
8. Publish approved evidence and the report through a recoverable, idempotent
   transition. Report publication states are `draft`, `publishing`,
   `ready_for_citation`, `rejected`, and `withdrawn`; ingestion status is separate.
   Only ready reports and their approved evidence are eligible for ordinary
   citations, regardless of vector-index contents.
9. Editing or regenerating creates a new draft version, never mutates an approved
   version. The previously approved version stays active until an explicit
   replacement or withdrawal. Retain report-to-evidence lineage and approval
   history under the documented retention/deletion policy.

Source approval means eligibility for retrieval, not proof that every source
claim is true. Approval does not publish unrelated staged acquisitions. Unapproved
source revisions remain staged even if their URL or bytes match approved content;
reuse must preserve owner and publication scope. A generated report must remain
distinguishable from primary evidence so it cannot corroborate itself in later
research.

## Ownership and proposed API

- `agentic-rag` identifies an evidence gap and requests a run through
  `knowledge-api`; it does not fetch URLs or mutate the corpus directly.
- `knowledge-api` owns authenticated run submission, report review, exact-version
  approval, withdrawal, and the publication boundary.
- `knowledge-worker` owns durable research orchestration, acquisition handoff,
  staged ingestion, report generation/verification, and publication work.
  Separate job kinds and dispatch explicitly: the existing ingest/delete worker
  must not claim research jobs as deletion work.
- Trawler owns public acquisition and SSRF enforcement. It must not receive
  browser sessions, Lighthouse bearer tokens, or database/vector credentials.
  Design a narrow, owner-bound asynchronous handoff; do not retain user session
  cookies in queued jobs or reuse read-only MCP tokens for writes.
- Chartroom owns the research toggle, destination selection, run progress,
  draft/evidence review, and explicit approval action.
- `knowledge-learning` remains unchanged; later feedback, freshness, and tuning
  may consume reviewed research outcomes.

Proposed owner-scoped routes, to finalize before implementation:

| Route | Purpose |
| --- | --- |
| `POST /v1/research-runs` | Submit a bounded, idempotent research request |
| `GET /v1/research-runs/{id}` | Inspect progress, failures, budgets, and draft link |
| `POST /v1/research-runs/{id}/cancel` | Cancel without discarding retained outcomes |
| `POST /v1/research-runs/{id}/retry` | Retry an exhausted failed run using retained evidence |
| `GET /v1/reports/{id}/versions/{version}` | Review one saved version and its evidence |
| `POST /v1/reports/{id}/versions/{version}/approval` | Approve the exact hash/version |
| `POST /v1/reports/{id}/versions/{version}/rejection` | Reject a draft with a recorded decision |
| `POST /v1/reports/{id}/versions/{version}/withdrawal` | Stop an approved version supporting new answers |

## Implementation sequence and exit gates

1. **Persistence and visibility first:** add research runs, separate research jobs,
   report versions, evidence associations, and approval records through additive
   migrations. Enforce owner and publication checks in search, questions, direct
   source access, and post-generation citation rechecks before enabling acquisition.
   Keep workload-only services inside their workload packages, one top-level
   class per module; retain `workload.py` as the composition entry.
2. **Manual-source vertical slice:** accept a small list of permitted public URLs
   in an explicitly submitted research run. Reuse Trawler for fetches, stage
   evidence, save a verified draft, and implement exact-version approval.
   This proves the workflow before adding autonomous source selection.
3. **Bounded automatic discovery:** integrate a configured search provider, evidence
   sufficiency routing, budgets, cancellation, durable retries, and duplicate-run
   handling. Use the same review/publication path as manual-source runs.
4. **Review UI and recovery:** wire Chartroom progress and approval; extend
   backup/restore, deletion, vector rebuild, readiness, and metrics to research
   data and staged artifacts. Deployment configuration remains a separate
   Infrastructure/Fleet handoff with no automatic production rollout.

Acceptance requires:

- The Azure Foundry scenario saves a cited draft, survives restart, and exposes
  exact supporting evidence for review.
- Before approval, draft text and staged sources never appear in ordinary search,
  answers, direct source/original access, or read-only MCP results.
- Approval makes only the named version and selected evidence eligible for
  citation; changed versions require fresh approval. Concurrent approval,
  rejection, withdrawal, and deletion cannot bypass publication checks.
- Cross-owner IDs, forged/stale hashes, injected source instructions, private
  URL targets, redirects, and budget exhaustion have explicit tested outcomes.
- Crashes between database and vector writes do not leak drafts; retries do not
  duplicate reports or publish an unintended version.
- A second question can cite the approved report with traceable primary evidence.
  Withdrawal/deletion prevents new citations, including during answer generation.
- Real-model evaluation checks summary support, coverage, provenance, uncertainty,
  and latency separately; the existing source-first acceptance gates still pass.
- Restore and index rebuild preserve both approval state and staging isolation.
  Deterministic fixtures prove the workflow; a live official-source Azure Foundry
  run is a separate acceptance check, not a network-dependent unit test.

## Implementation progress

The local Lighthouse implementation now includes the first manual-source slice:

- Additive `0003` migration for owner-scoped runs, isolated research jobs,
  immutable report versions, source lineage, and decision history.
- One-to-three explicitly supplied HTTPS URLs are acquired by the durable
  knowledge-worker research queue through Trawler's existing single-URL API.
  Staged Lighthouse document IDs are reconciled back to Trawler's retained
  acquisition handoff record on retries.
  URL policy, DNS, redirects, MIME type, and byte/time limits remain Trawler's
  responsibility. No browser cookie or Lighthouse credential is sent to Trawler.
- Acquired normalized text and provenance are persisted as staged evidence.
  Staged documents have no active retrieval version or vector points, and
  ordinary search, answers, document listing/status, direct source, and original
  access recheck publication state.
- The worker creates a versioned, hash-addressed draft with source IDs and
  URLs, model identity, and an explicit unverified/uncertainty notice. Approval
  requires the exact version hash, records the owner and timestamp, and queues
  idempotent report and evidence publication. Approved reports have their own
  indexed documents; citations include report identity, version, and supporting
  chunk IDs to distinguish generated text from primary sources.
  Rejection and withdrawal are recorded;
  withdrawal hides evidence in PostgreSQL before vector cleanup.
- Artifact cleanup addresses are saved before object writes. Failed evidence
  storage retries reuse the retained Trawler acquisition without refetching;
  partial vector publication remains invisible until PostgreSQL admits the exact
  version. Deleting supporting evidence invalidates report citations.
- Backup quiescence and original verification include research jobs and staged
  originals; reindex includes only published evidence and can rebuild approved
  research documents from their normalized text. These recovery paths have
  deterministic local checks; an actual backup/restore round-trip remains to
  be verified.

This is not yet autonomous discovery or a complete release. The summary uses
the configured structured answer model and constrains citations to supplied
chunks, but does not independently verify every generated claim; human review
is mandatory. Research has no Chartroom UI, search-provider discovery,
replacement-version workflow, or production runtime wiring. Those remain
separate implementation and acceptance gates.

### Local validation (2026-11-05)

- Full Lighthouse suite: **290 passed, 10 skipped**, using SQLite and fake
  model/vector/storage dependencies for the new workflow. PostgreSQL-specific
  and opt-in live integrations were not exercised in this run.
- Tests cover staged-source isolation, exact-hash approval, approved report
  retrieval with primary-evidence lineage, rejection, withdrawal, cross-owner
  access, idempotent submission, cancellation, expired leases, retained-source
  storage retries, partial vector-write recovery, reindexing, and citation
  invalidation after supporting-evidence deletion.
- Source distribution and wheel builds passed; the API contract was regenerated.
- No production mutation or rollout was performed. Live Trawler/model acceptance
  and a real research-aware backup/restore round-trip remain outstanding.
