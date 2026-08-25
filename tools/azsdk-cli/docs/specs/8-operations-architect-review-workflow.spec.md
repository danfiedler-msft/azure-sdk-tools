# ARH Replacement: Decisions and Open Design Areas

> **Status:** Working draft for design alignment. This document identifies the
> decisions required before API Review Hub (ARH) can replace the Azure SDK review
> issue workflow and APIView-based approval model. It intentionally does not
> re-document every current process mechanic.

---

## Goal

Replace the existing Azure SDK Architecture Board workflow - SDK review issues,
approval labels, and APIView coordination - with one ARH-centric approval experience
that:

- provides a single source of truth for SDK API approvals;
- gives architects a unified work queue across repositories;
- removes duplicate review tracking;
- supports generated and manually authored SDKs;
- preserves cross-language service context; and
- satisfies release-gating and audit requirements.

The emerging end state is:

```mermaid
flowchart LR
    START["Review initiation signal (TBD)"] --> REQUEST["Canonical review trigger"]
    REQUEST --> ARH["ARH creates or updates review PR"]
    ARH --> REVIEW["Architect reviews API diff"]
    REVIEW --> RECORD["ARH records decision against API hash"]
    RECORD --> BOARD["ARH projects state to architect dashboard"]
    RECORD --> GATE["Release gate verifies exact API hash"]
    GATE --> RELEASE["SDK release"]
```

The trigger, retirement of SDK review issues, service-level grouping, and release
integration are not yet settled. Those decisions are the focus of this document.

---

## Definitions

- **Working PR:** The pull request in an `azure-sdk-for-<language>` repository
  containing the SDK intended to merge and release.
- **Review PR:** An ARH-managed pull request containing the reviewable API diff.
  Although its synthetic branches can technically be merged, doing so has no effect
  on an SDK codebase and both branches are eventually deleted. The expected workflow
  is to close the review PR rather than merge it.
- **API hash:** The identifier for the exact API surface reviewed. Approval of one
  hash does not approve a changed surface.
- **Projection:** GitHub labels or project fields that communicate and filter ARH
  state. The approval action originates in GitHub, ARH stores the resulting decision
  against the API hash, and ARH projects labels back to GitHub. The projection is
  not the approval record.
- **SDK review issue:** The current coordinating issue in `Azure/azure-sdk` used for
  intake, artifact validation, language grouping, assignment, and approval tracking.
- **Canonical service identity:** A stable identifier used to correlate language
  reviews for one service or release. It is not merely a display name in a PR title.

---

## Decisions at a glance

| # | Design area | Emerging direction | Decision still required |
|---|-------------|--------------------|-------------------------|
| 1 | Source of truth | ARH record bound to API hash | Migrate APIView release consumers and inventory informational label consumers |
| 2 | SDK review issues | Retire after capability parity | Confirm whether any intake or coordination purpose remains |
| 3 | Architect dashboard | [Azure organization project POC](https://github.com/orgs/Azure/projects/1018/views/4) backed by ARH | Validate fields, filtering, grouping, and ownership |
| 4 | Review trigger | No canonical trigger selected | Decide among explicit and automated initiation paths |
| 5 | SDK coverage | ARH is independent of SDK origin | Define how non-generated SDK reviews are initiated |
| 6 | Review artifact | ARH review PR associated with a working PR | Confirm APIView retirement and review PR lifecycle |
| 7 | Approval semantics | GitHub activity stored in ARH and projected back to GitHub | Finalize labels and qualifying GitHub activity |
| 8 | Release integration | Gate on exact approved API hash | Define when approval is required and which system enforces it |
| 9 | Governance boundary | ARH stores SDK API and package-name approvals | Define GitHub package-name actions and interfaces to other boards |

---

## 1. Source of truth

### Decision needed

What is the authoritative approval record?

### Current state

- APIView is the authoritative SDK API approval record.
- Release gates consult APIView. They do not consult SDK review issue labels,
  GitHub review state, or Azure DevOps work items.
- SDK review issues display per-language labels such as `<lang>-api-approved`, but
  those labels are informational for service teams.

### Proposed direction

ARH becomes authoritative. It records the architect, decision, language, timestamp,
working PR association, and API hash. GitHub labels and project fields become
projections of that record.

```text
ARH approval record (authoritative)
        |
        +--> working PR label (projection)
        +--> review PR label (projection)
        +--> project field (projection)
        +--> release-gate response
```

### Required decisions

- [ ] Inventory APIView consumers and any dashboards, bots, or workflows that use
  informational approval labels.
- [ ] Decide the transition contract for release gates moving from APIView to ARH.
- [ ] Define stale-approval invalidation when the working API hash changes.
- [ ] Define ARH availability and failure behavior for a blocking release gate.
- [ ] Confirm the audit-retention requirements for approval records.

---

## 2. Fate of Azure SDK review issues

### Decision needed

Can the `Azure/azure-sdk` SDK review issue workflow be completely retired?

### Capabilities that must be replaced

| Current issue capability | Candidate replacement |
|--------------------------|-----------------------|
| Intake | Canonical ARH trigger |
| Artifact validation | ARH validates working PR and API artifact |
| Cross-language grouping | Canonical service identity on the dashboard |
| Review readiness | ARH review state |
| Architect assignment | ARH routing configuration |
| Approval visibility | ARH state projected to the issue or dashboard |
| Completion | ARH closes or archives review artifacts according to policy |
| Scheduling / Bookings | Release-readiness integration or explicit fallback |
| Audit history | ARH record plus review PR conversation |

### Proposed direction

Retire the issue workflow once ARH and the dashboard have demonstrated capability
parity. The issue has never been an authoritative approval source; during migration
it may remain only as a compatibility coordination view. APIView remains
authoritative until the release gate cuts over to ARH.

### Required decisions

- [ ] Does any stakeholder still need an issue as an intake or communication
  artifact?
- [ ] Can ARH preserve the service-level context currently supplied by one
  multi-language issue?
- [ ] What is the parity period and what measurable result ends it?
- [ ] What happens to Bookings and already scheduled reviews?
- [ ] Are historical issues retained as read-only records?

This is the central replacement decision: if the issue retains intake or grouping
responsibility, the target experience still has a parallel coordination process.

---

## 3. Architect dashboard experience

### Decision needed

What becomes the architect's primary work queue?

### Proposed direction

The [Azure SDK Architecture Board ARH view
POC](https://github.com/orgs/Azure/projects/1018/views/4) becomes the review queue,
ownership view, and status view across ARH review PRs. It is a view over ARH state,
not a second workflow engine.

The dashboard must answer:

- What needs my review?
- Which language, package, service, and release is it for?
- Is it awaiting review, awaiting service-team changes, or approved?
- Where are the review PR and working PR?
- Which other language reviews belong to the same service release?

### Required filtering and grouping

| Need | Proposed data |
|------|---------------|
| Filter by architect | ARH-configured reviewer or assignee |
| Filter by language | Language field |
| Filter by approval state | ARH-synchronized review-state field or label |
| View by service | Canonical service identity |
| Group related languages | Service identity plus release/correlation ID |
| Navigate to implementation | Working PR association |
| Correlate one release internally | Release or review correlation ID managed by automation |

### Service grouping decision

ARH creates one review item per language, while the current issue groups all
languages. A PR title convention or service-name label is useful for display but is
not a durable grouping key.

Preferred contract:

```text
canonical-service-id + release-correlation-id
    -> language
        -> package/namespace
            -> review PR
```

Release-plan metadata may supply this correlation internally, but architects should
not need to open or see Azure DevOps release-plan work items.

### Required decisions

- [ ] What system owns the canonical service identity: release automation, Service
  Tree, package metadata, or ARH?
- [ ] Which Azure-owned repository hosts production ARH review PRs so the
  organization project can index them?
- [ ] Are project fields synchronized by ARH, or are labels the only routing input?
- [ ] How are existing open review PRs backfilled?
- [ ] Does the project show only SDK Architecture Board work or link to the other
  architect boards as separate views?

The project `Done` status must not be interpreted as API approval. Approval remains
an ARH decision against an API hash.

---

## 4. Review trigger model

### Decision needed

What single event causes ARH to create or update a review?

### Options discussed

| Option | Advantages | Risks |
|--------|------------|-------|
| Working PR label | Simple, explicit, works across SDK origins | Manual and inconsistently applied |
| Release planner guidance plus Azure SDK agent query | Carries release and service context | Dashboard is read-only; service team must manually run the suggested agent query |
| `azsdk` command | Explicit and usable from local or agent workflows | Discoverability and training burden |
| Automated release-readiness trigger | Lowest service-team effort and aligns with release preparation | Signal and coverage across release paths are not yet defined |

### Proposed direction

No canonical trigger has been selected. An automated release-readiness trigger should
be evaluated alongside the explicit options above. If selected, it must occur before
merge, support normal release paths, and retain one explicit fallback for exceptional
workflows.

### Required decisions

- [ ] What exact pre-merge event represents release readiness?
- [ ] If automation is selected, is the fallback a working PR label, release
  planner-guided agent query, or `azsdk` command?
- [ ] Who is authorized to request, cancel, or restart a review?
- [ ] Does a new commit update the existing review or create a new review?
- [ ] How are abandoned and superseded requests detected?

One canonical trigger does not mean one transport. Automation and an explicit
fallback may call the same idempotent ARH request contract.

---

## 5. Review initiation across SDK origins

### Decision needed

How is an ARH review initiated for each SDK path?

ARH itself is already independent of TypeSpec and the release agent. The remaining
gap is the caller and trigger that create or update an ARH review for each path,
especially manually authored and locally generated SDK PRs.

### Required scenarios

- Management-plane SDK PR generated automatically after spec merge.
- Data-plane SDK generated on demand.
- SDK PR generated locally.
- Hand-authored or non-TypeSpec SDK.
- Customization-only or dependency release with no spec change.
- Storage-style or other team-owned handoff workflow.

### Existing ARH contract

ARH starts from a working SDK PR and API artifact, not from a TypeSpec project:

```text
repository + working PR + language + package/namespace
    + canonical service identity + API artifact/hash
    + optional release plan
```

Generation-specific systems can invoke this same contract. They do not require
separate approval models.

### Required decisions

- [ ] What invokes ARH today for manually authored and locally generated SDK PRs?
- [ ] Which canonical trigger should invoke ARH for those paths in the target flow?
- [ ] What metadata is mandatory when no release plan exists?
- [ ] Does any SDK path still lack a deterministic API artifact?

---

## 6. Review artifact model

### Decision needed

What artifact do architects review, and how is it kept aligned with the working PR?

### Current state

APIView presents generated API surface revisions in a separate web experience.
Architects often review autorevisions after merge rather than a working PR.

### Proposed direction

ARH creates or updates a synthetic review PR containing an `API.md` diff associated
with the working PR. Architects review that PR for SDK API approval. The review PR
is closed rather than merged because merging its temporary synthetic branches does
not change the SDK codebase.

### Required decisions

- [ ] Does APIView fully disappear after migration, or does any scenario still
  require it?
- [ ] Does each working PR map to one durable review PR that updates as the API
  changes?
- [ ] What identifies the baseline used for the API diff?
- [ ] What closes the review PR: approval, working PR merge, release, supersession,
  or abandonment?
- [ ] How are review comments preserved when the diff changes?

The intended association is one durable ARH review PR for the working SDK PR. As the
SDK API changes, ARH updates that review rather than creating disconnected review
artifacts, keeping feedback associated with the implementation intended to ship.

---

## 7. Approval semantics

### Decision needed

Which states exist, who can cause them, and how are they represented?

### Proposed states

| ARH state | Proposed projection label | Meaning |
|-----------|---------------------------|---------|
| Review needed | `api-review-needed` | Active API hash awaits architect review |
| Changes requested | `api-changes-requested` | Authorized architect requested changes |
| Approved | `api-approved` | Active API hash is approved |

Requirements:

- only one projected state is present at a time;
- qualifying GitHub review activity is interpreted and stored by ARH against the
  active API hash;
- ARH manages labels on both review and working PRs;
- labels are informational and are never the release source of truth;
- a new API hash invalidates `api-approved`;
- native GitHub approval counts only when performed by an authorized architect; and
- implementation approval from another maintainer does not imply API approval.

### Required decisions

- [ ] Are these the final cross-repository label names?
- [ ] Do existing language repositories, especially JavaScript, use conflicting
  labels or semantics?
- [ ] Does `changes requested` come directly from native GitHub review state, an ARH
  action, or both?
- [ ] How are architect groups, substitutes, and unauthorized decisions configured
  and audited?
- [ ] Must labels exist on both review and working PRs, or is one location enough
  after dashboard synchronization?

---

## 8. Release-readiness integration

### Decision needed

When is SDK Architecture Board approval required, and which system enforces it?

### Proposed direction

Review is requested when a release is being prepared, and the release gate asks ARH
whether the exact API hash being published has an applicable approval.

This separates three events:

1. a working SDK PR becomes release-ready;
2. an architect approves an API hash; and
3. a release pipeline verifies that approval before publication.

### Required decisions

- [ ] Is approval required before working PR merge, before release execution, or
  both?
- [ ] Which release types require architecture review: first preview, preview
  update, first GA, GA update, and patch?
- [ ] What system computes the release API hash?
- [ ] Does the release pipeline temporarily dual-read APIView and ARH?
- [ ] What ends the dual-read migration period?
- [ ] What is the fail-closed behavior when ARH is unavailable?

---

## 9. Relationship with the three-board model

Laurent's process model identifies three independent governance concerns:

1. Breaking Change Board
2. Stewardship Board
3. SDK Architecture Board

This replacement effort concerns only the SDK Architecture Board workflow.

### Proposed boundary

| ARH owns or records | ARH does not own |
|---------------------|------------------|
| SDK public API review | Stewardship review of data-plane specifications |
| SDK API review artifact | Breaking-change approval |
| SDK architect assignment and decision | ARM or spec-level governance |
| API-hash approval record | General SDK implementation approval |
| Package-name approval record relayed from GitHub | The GitHub action where an architect approves a package name |
| SDK Architecture Board work queue | Release approval unrelated to API or package-name review |

### Boundary requiring clarification

Package name review remains an action on the GitHub spec PR. The emerging direction
is for that action to be relayed to and stored in ARH, then exposed under the same
release gate as SDK API approval. The dashboard may expose package-name work as a
distinct view while ARH provides one release-facing approval store.

---

## End-state workflow requiring alignment

```mermaid
sequenceDiagram
    participant S as Service team / automation
    participant W as Working SDK PR
    participant A as ARH
    participant R as Review PR
    participant D as Architect dashboard
    participant X as Architect
    participant G as Release gate

    S->>W: SDK working PR is available
    S->>A: Review initiation signal (TBD)
    A->>R: Create or update API diff
    A->>D: Add item, identity, owner, and state
    X->>R: Approve or request changes
    A->>A: Record decision against API hash
    A->>W: Project review state
    G->>A: Verify release API hash
    A-->>G: Approved / not approved
```

The target workflow is aligned in principle. The following are the primary design
review topics:

1. canonical trigger and fallback;
2. complete retirement criteria for SDK review issues;
3. canonical service identity and cross-language grouping;
4. initiation path for manually authored and locally generated SDKs;
5. final approval-state and label contract;
6. release types and enforcement point; and
7. boundary of package-name work within the architect experience.

---

## Implementation sequence

### Phase 1: Resolve contracts

- Inventory APIView release consumers and informational approval-label consumers.
- Select the trigger and fallback.
- Define canonical service identity.
- Finalize approval states and label names.
- Define release requirements by release type.
- Define how GitHub package-name approval is relayed to ARH.

### Phase 2: Dashboard and association proof of concept

- Host review PRs in an Azure-owned repository.
- Persist links among review PR, working PR, language, package, service identity,
  and release plan.
- Populate the Azure SDK Architecture Board project.
- Validate filtering, assignment, state, and cross-language grouping.
- Backfill existing open ARH reviews.

### Phase 3: Approval and release integration

- Enforce architect authorization.
- Record and invalidate decisions by API hash.
- Synchronize projections to GitHub and the project.
- Integrate the exact-hash release check.
- Exercise TypeSpec, non-TypeSpec, local, and hand-authored paths.

### Phase 4: Retire parallel workflow

- Run a time-boxed comparison against current SDK review issues.
- Measure missing reviews, routing errors, stale state, and label drift.
- Stop creating new issues after parity criteria pass.
- Preserve historical issues without retaining approval authority.
- Remove APIView and label consumers only after their replacement is verified.

---

## Success criteria

- [ ] ARH is the only authoritative SDK Architecture Board approval store.
- [ ] Every release path can submit a working PR and deterministic API artifact.
- [ ] Every active review appears in the architect dashboard with owner, language,
  service, package, working PR, review PR, and state.
- [ ] Related language reviews group without title parsing.
- [ ] Approval is accepted only from authorized architects and binds to the exact API
  hash.
- [ ] Changing the API invalidates stale approval automatically.
- [ ] The release gate verifies ARH approval for the exact hash being published.
- [ ] The SDK review issue workflow can be disabled without losing intake,
  validation, grouping, assignment, approval, or audit behavior.
- [ ] APIView can be removed without leaving an unsupported review path.
- [ ] Stewardship and breaking-change governance remain independent.

---

## Validation and metrics

Validate the design across management plane, data plane, TypeSpec, no-spec-change,
locally generated, and hand-authored SDK scenarios.

| Metric | Purpose |
|--------|---------|
| Review request to first architect action | Review latency |
| Changes requested to updated API hash | Service-team response time |
| Stale approvals invalidated | Hash-binding effectiveness |
| Dashboard items missing associations | Integration correctness |
| Reviews without canonical service identity | Grouping coverage |
| Manual label corrections | Projection drift |
| Abandoned review PRs | Trigger-timing quality |
| New SDK review issues after cutover | Retirement completeness |
| Release-gate label reads after cutover | Source-of-truth migration completeness |

Do not collect API contents, credentials, or customer data. API hashes, repository
and PR identifiers, workflow state, and timings are sufficient.

---

## References

- [TypeSpec-to-SDK Release Workflow](typespec-to-sdk-release-workflow.spec.md)
- [Architect review component map](https://gist.github.com/samvaity/870950dec333779cc9fe28d87c3ad703)
- [Azure SDK Architecture Board project](https://github.com/orgs/Azure/projects/1018)
- [Data-plane label consolidation](https://github.com/Azure/azure-rest-api-specs/issues/45437)
- [`azsdk` ARH behavior when no API hash is provided](https://github.com/Azure/azure-sdk-tools/pull/16773)
