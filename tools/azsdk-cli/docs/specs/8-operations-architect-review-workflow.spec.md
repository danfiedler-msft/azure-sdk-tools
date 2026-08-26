# ARH as the Shared Record for SDK Architecture Approvals

> **Status:** Working draft. This document separates established behavior from the
> decisions still needed before API Review Hub (ARH) can replace APIView and SDK
> review issue tracking.

---

## Goal

Use ARH as the shared record for SDK API and package-name approvals.

The approval actions remain in the GitHub PR where each review happens:

- Architects approve the SDK API on the ARH review PR.
- Architects approve package names on the `azure-rest-api-specs` PR.

ARH stores both decisions and makes their status available to architect dashboards
and release pipelines. How package-name decisions reach ARH is still open.

The new process should:

- keep release-facing approval records in one place;
- give architects a clear review queue;
- remove duplicate tracking in SDK review issues;
- work for generated and manually written SDKs;
- group related language packages under one service;
- show package names that need approval and names approved earlier; and
- block release when required approval is missing.

```mermaid
flowchart LR
    START["SDK API review request (trigger TBD)"]
    START --> REVIEWPR["ARH creates or updates a review PR"]
    REVIEWPR --> APIREVIEW["Architect decides on the API"]
    APIREVIEW --> APIRECORD["ARH stores the API decision"]

    SPECPR["Spec PR changes package names"]
    SPECPR --> NAMEREVIEW["Architect approves names on the spec PR"]
    NAMEREVIEW --> NAMERECORD["ARH stores the package-name decision (integration TBD)"]

    APIRECORD --> BOARD["Project view 4 lists and filters ARH review PRs"]
    NAMERECORD --> NAMEVIEW["Package-name dashboard or view (TBD)"]
    APIRECORD --> GATE["Release pipeline checks required approvals"]
    NAMERECORD --> GATE
    GATE --> RELEASE["SDK is released"]
```

---

## Terms

- **Working PR:** The SDK pull request that will be merged and released.
- **Review PR:** The pull request created by ARH for reviewing the SDK API. It uses
  temporary branches and does not change the SDK repository.
- **API hash:** An ID for the exact SDK API that was reviewed. If the API changes,
  the old approval no longer applies.
- **SDK review issue:** The current issue in `Azure/azure-sdk` used to collect
  review links, group languages, and show review status.
- **ARH Service and Package:** ARH groups related Packages under a Service.
- **Status label:** A GitHub label such as `api-approved` that shows ARH state. The
  label is not the approval record used for release.

---

## Decisions at a glance

**Priority**

- **P0:** Must be decided before the old review process can be removed.
- **P1:** Needed for the complete architect experience, but does not block basic ARH
  approval storage and release checks.

| # | Priority | Decision | What is already known |
|---|----------|----------|-----------------------|
| [1](#decision-1) | P0 | Choose how SDK API review starts for every SDK path. | All paths can call the existing ARH create operation. |
| [2](#decision-2) | P0 | Decide how package-name approval is recorded in ARH and when old approval can be reused. | Approval remains on the GitHub spec PR; ARH stores the result for release. |
| [3](#decision-3) | P0 | Decide whether SDK review issues can be removed and how meeting requests work afterward. | ARH already groups Packages under a Service and stores API decisions. |
| [4](#decision-4) | P1 | Finish the architect dashboard design, including package-name work. | [Project view 4](https://github.com/orgs/Azure/projects/1018/views/4) is the ARH review PR queue. |
| [5](#decision-5) | P1 | Finalize SDK API status labels and architect authorization. | GitHub review activity triggers ARH; ARH stores the decision and updates labels. |

---

## Approval authority and source of truth

### Today

- APIView is the SDK API approval record used by release pipelines.
- SDK review issue labels such as `<lang>-api-approved` are informational.
- Package-name approval happens on the `azure-rest-api-specs` PR. Protected labels
  and a required GitHub check block the spec PR until required architects approve.

### Target

ARH becomes the shared record used by release pipelines.

For SDK API approval:

- The architect decides on the ARH review PR.
- ARH stores the API hash, decision, architect, language, package, working PR, and
  decision time.

For package-name approval:

- The architect decides on the `azure-rest-api-specs` PR.
- The existing workflow verifies the architect.
- ARH stores the Service, language, exact package name, architect, decision, and
  decision time.
- The architect does not approve the package name again in ARH.

SDK API approval applies only to its API hash. A changed SDK API has a new hash and
requires approval.

Questions answered by the current design:

- **Question:** What is the authoritative SDK API approval record today?

  **Answer:** APIView. SDK review issue labels are informational, and release does
  not consult GitHub or Azure DevOps for SDK API approval.

- **Question:** How do release pipelines transition from APIView to ARH?

  **Answer:** The new release components use `azsdk` commands that query APIView or
  ARH. This provides compatibility while languages onboard to ARH.

- **Question:** How does ARH prevent approval from applying after the API changes?

  **Answer:** The language repository submits the exact API hash. A changed surface
  produces a different hash, so the approval check fails. ARH can show approved
  hashes to a person, but CI still fails for the unapproved hash.

- **Question:** Does APIView also gate directly on the submitted API hash?

  **Answer:** Not directly. The APIView release check finds a matching package and
  version. APIView uses the hash internally to copy approval from an older version
  when the API is unchanged. ARH instead requires the release check to submit the
  exact hash.

---

<a id="decision-1"></a>

## Decision 1 (P0): How does SDK API review start?

No normal trigger has been selected.

Today, authorized CoreAI users can call `azsdk api-review create`. The command
requires:

- language;
- package name;
- target branch;
- optional base tag or ref;
- target owner, which defaults to `Azure`; and
- target repository, which ARH can choose from the language.

Calling ARH again for the same package, base, and target returns the existing review
instead of creating a duplicate.

| Option | Benefit | Drawback |
|--------|---------|----------|
| Label on the working SDK PR | Simple and works for any SDK PR | Someone must remember to add it |
| Guidance in the release planner | Includes service and release information | The dashboard is read-only; the service team must copy and run an Azure SDK agent request |
| `azsdk` or Azure SDK agent request | Explicit and works outside the release planner | Service teams must know what to run |
| Automatic release-readiness event | Requires the least service-team work | The event and covered SDK paths are not defined |

The decision must cover:

- [ ] Which option is the normal trigger?
- [ ] If it is automatic, which event starts it?
- [ ] What is the manual backup?
- [ ] What invokes ARH for manually written or locally generated SDK PRs?
- [ ] Do management plane, data plane, no-spec-change, and team-specific handoff
  flows use the same trigger?
- [ ] What does ARH do if a review PR is closed before a decision, and what happens
  when the same review is requested again?

Questions answered by the current implementation:

- **Question:** Who is authorized to request a review?

  **Answer:** Anyone in the CoreAI security group can reach the create endpoint.

- **Question:** Who is authorized to cancel or restart a review?

  **Answer:** Cancel and restart are not ARH operations.

- **Question:** Who can close a review PR?

  **Answer:** Anyone with the required GitHub repository permission.

- **Question:** Does a new commit update the existing review or create another one?

  **Answer:** A target-branch commit updates the existing review when it changes the
  API diff. It never creates another review.

- **Question:** What happens when the same package, base, and target are requested
  again?

  **Answer:** The create endpoint returns the existing review PR instead of creating
  a duplicate.

- **Question:** Is ARH limited to TypeSpec or release-agent SDKs?

  **Answer:** No. ARH is independent of TypeSpec and the release agent. Generated,
  manually written, and locally generated paths can use the same create endpoint.

- **Question:** What information does a caller have to provide?

  **Answer:** Language, package name, and target branch are required. The caller can
  also provide a base tag or ref and target owner. ARH can infer the repository from
  the language.

- **Question:** Who produces the API artifact and hash for generated and
  non-generated SDKs?

  **Answer:** The language repository controls artifact generation and defines the
  API hash for its language. ARH is not dependent on how the SDK was produced.

- **Question:** Who owns failures that prevent creation of a reviewable artifact?

  **Answer:** The ARH team owns failures in ARH or shared language-independent
  templates. The language team owns failures in language-specific components.

All entry points should use the existing create operation. The remaining question
is how ARH treats a review PR that GitHub permits someone to close before the review
is complete.

---

<a id="decision-2"></a>

## Decision 2 (P0): How is package-name approval recorded in ARH?

### Current GitHub process

The [package-name review
process](https://github.com/samvaity/azure-rest-api-specs/blob/bbd22cbebd12870570ea5795ff10c45ae5b83522/.github/workflows/src/package-name-approval/PACKAGE-NAME-REVIEW-PROCESS.md)
runs on the spec PR:

- A workflow finds new or changed package names in `tspconfig.yaml`.
- If any name changes, all Tier 1 language architects approve the cross-language
  name set.
- The PR comment shows changed names, configured but unchanged names, and languages
  not yet configured.
- Architects approve with protected GitHub labels.
- A required check blocks merge until all required approvals are present.
- If a name changes again on the same PR, only that language's approval is removed.

This process blocks the spec PR and remains separate from later SDK API review.

### Intended ARH result

The architect continues to approve on the GitHub spec PR. ARH stores the verified
result, and `azsdk package get-approval-status` returns it to the release pipeline.
The mechanism that records the GitHub decision in ARH is not designed.

Possible recording signals are:

- each verified `package-name-<lang>-approved` label;
- the aggregate `package-name-approved` state;
- the final approval-label state when the spec PR merges; or
- GitHub events received directly by the ARH GitHub App.

Waiting for merge would let ARH record the result for release, but the current
GitHub check would remain responsible for blocking the spec PR.

If old approval is reused, the GitHub table could show:

| ARH state | Display | Architect action |
|-----------|---------|------------------|
| New, changed, or not approved | Pending | Review and approve on the spec PR |
| Exact name approved earlier | Approved, greyed out | No action unless the full cross-language set must be confirmed again |
| Language not configured | Not configured, greyed out | Decision needed |

The decision must cover:

- [ ] Which event records approval in ARH?
- [ ] Does ARH participate in the spec PR merge check, or only store the result for
  release?
- [ ] What fields identify reusable approval: Service, language, and exact package
  name, or more?
- [ ] How are approvals created before ARH imported?
- [ ] When one language's name changes, must all Tier 1 languages approve again?
- [ ] If they must, how does a greyed-out old approval show that confirmation is
  still needed?
- [ ] What does an architect approve for a Tier 1 language not yet configured?
- [ ] Does ARH keep an old name's approval as history after the name changes?
- [ ] Are protected labels still the GitHub approval action? If not, what replaces
  them?
- [ ] How does the package-name process find the correct ARH Service and Package?

Questions answered by the proposed integration:

- **Question:** Where does package-name approval happen, and where is it stored?

  **Answer:** The architect acts in GitHub on the spec PR. ARH records the verified
  decision.

- **Question:** How does the release pipeline read package-name approval?

  **Answer:** It queries ARH through `azsdk package get-approval-status`, alongside
  SDK API approval.

- **Question:** Does a separate package-name dashboard prevent ARH from storing the
  approval?

  **Answer:** No. A separate Project view can present the work while ARH remains the
  release-facing record.

---

<a id="decision-3"></a>

## Decision 3 (P0): Can SDK review issues be removed?

The issue currently:

| Purpose | ARH replacement |
|---------|-----------------|
| Starts review | Decision 1 trigger |
| Collects review links | ARH links the working PR and review PR |
| Groups languages | ARH Service with child Packages |
| Assigns architects | ARH reviewer assignment |
| Shows status | ARH labels and Project view |
| Keeps discussion history | ARH review PR |
| Supports Bookings and meetings | Not decided |

The replacement must continue showing:

- service name;
- packages and languages in the review;
- working and review PRs;
- assigned architects;
- status for each package and language;
- supporting links; and
- meeting request and meeting details when needed.

The decision must cover:

- [ ] Is an issue still needed to request review or communicate with the service
  team?
- [ ] Do reviews already scheduled through Bookings finish through the old process
  or move into ARH?
- [ ] How does a service team request a live architect meeting after issues are
  removed?
- [ ] Where are the meeting date, link, agenda, packages, and outcome recorded?
- [ ] Is the final decision after a meeting always stored in ARH?
- [ ] Should old issues remain as read-only history?
- [ ] How long do both processes run before new issues stop?

Questions answered by the current models:

- **Question:** Did the SDK review issue provide an authoritative approval?

  **Answer:** No. The issue never had approval capability; its labels were
  informational.

- **Question:** Can ARH preserve the service-level grouping currently shown by one
  multi-language issue?

  **Answer:** Yes. ARH has a top-level Service with child Packages, so related
  language packages can be grouped under one service.

If the issue remains required for intake or communication, teams still have two
places to track one review.

---

<a id="decision-4"></a>

## Decision 4 (P1): What appears on the architect dashboard?

The [ARH Project view](https://github.com/orgs/Azure/projects/1018/views/4) is the
cross-repository queue for ARH review PRs. Each item is an ARH review PR from a
language repository.

The proposed labels show review state:

| Review PR label | Meaning in the Project view |
|-----------------|-----------------------------|
| `api-review-needed` | Waiting for an architect |
| `api-changes-requested` | Waiting for service-team changes |
| `api-approved` | Approved |

Review PR labels support Project filtering. Working PR labels show the same state to
the service team but are not approval records. The Project `Status` field, such as
`Done`, is not approval and should be hidden if it adds no separate value.

ARH already groups Packages under a Service. A proposed display is to put the
Service name in each review PR title for text filtering. If titles are used, ARH
must correct manual edits.

Architects should not need to open Azure DevOps release-plan work items.

The decision must cover:

- [ ] Is the Service shown in the review PR title, a protected label, or a Project
  field?
- [ ] How are package-name reviews linked to the ARH Service?
- [ ] Are package-name reviews another view in this Project or a separate queue?
- [ ] What package-name status is shown: pending, approved earlier, or not
  configured?

Questions answered by the current dashboard design:

- **Question:** Where are production ARH review PRs hosted?

  **Answer:** In the language repositories.

- **Question:** How can architects filter by language?

  **Answer:** The repository can serve as the language because each language has its
  own repository.

- **Question:** Do architects need Azure DevOps release-plan work items?

  **Answer:** No. Release plans are for service teams and should not be required for
  architect review.

- **Question:** Can the Service name be used to group or filter review PRs?

  **Answer:** Yes. Each Package has a Service parent, and the unique Service name can
  appear in review PR titles for Project text filtering. If titles are used, manual
  edits need anti-tampering handling.

- **Question:** How can multiple reviews for the same Service be distinguished
  without relying on release plans?

  **Answer:** The suggested correlation is
  `service_id + package_version_api_version`. Architects should not need
  release-plan data.

- **Question:** Are labels or Project fields used for review-state filtering?

  **Answer:** Review PR labels are the current routing and filtering mechanism.

- **Question:** Does the Project `Done` status mean API approval?

  **Answer:** No. Approval is shown by `api-approved`; the `Done` field should be
  removed if it has no other meaning.

- **Question:** How are existing review PRs added to the Project?

  **Answer:** The current proof of concept requires them to be added manually.

---

<a id="decision-5"></a>

## Decision 5 (P1): What SDK API labels and architect rules are used?

SDK API approval happens on the ARH review PR. GitHub review activity triggers an
ARH webhook. ARH checks the reviewer, stores the decision for the API hash, and
updates labels.

Only one state label should be present. If the API hash changes, `api-approved` is
removed.

A normal SDK implementation approval is not an API approval. ARH counts only
decisions from an authorized architect on the ARH review PR.

The decision must cover:

- [ ] Are `api-review-needed`, `api-changes-requested`, and `api-approved` the final
  label names?
- [ ] Does any language repository already use those names differently?
- [ ] Where is the authorized architect list stored?
- [ ] How are backup reviewers handled?

Questions answered by the current implementation:

- **Question:** Where does `changes requested` come from?

  **Answer:** Native GitHub review activity triggers an ARH webhook. ARH responds to
  that state and manages the labels.

- **Question:** Are labels needed on both the review PR and working PR?

  **Answer:** Yes. Review PR labels support architect dashboard filtering. Working
  PR labels communicate the state to the service team.

---

## Established ARH review PR behavior

ARH keeps one review PR for the same package, base, and target:

- Repeating the same create request returns the existing review PR.
- A target-branch commit updates the review only when the SDK API diff changes.
- ARH never creates another review for that same request.
- The request can specify the release tag or ref used for comparison.
- GitHub preserves review comments.
- A person with GitHub permission can close the review PR.
- Merging the temporary review branches has no effect on SDK code, so the review PR
  should be closed instead.

Questions answered by the current implementation:

- **Question:** Can the review PR be merged?

  **Answer:** Technically yes, but it only merges the review branch into a synthetic
  base branch. It does not change SDK code, and both temporary branches are
  eventually deleted.

- **Question:** Does each target branch keep one review PR as feedback is addressed?

  **Answer:** Yes. Changes pushed to the target branch update the existing review PR
  when they change the API diff.

- **Question:** What identifies the baseline for the API diff?

  **Answer:** The caller supplies the base tag or ref. A skill or agent may infer it
  on the caller's behalf.

- **Question:** How are review comments preserved when the API diff changes?

  **Answer:** GitHub preserves them on the existing review PR.

- **Question:** Does APIView remain after ARH migration?

  **Answer:** No. APIView is deleted after every language is onboarded to ARH.

---

## Established release pipeline

[#16660](https://github.com/Azure/azure-sdk-tools/pull/16660) defines the release
check:

1. Each language pipeline runs `azsdk package get-approval-status` at its existing
   approval-check step.
2. ARH-enabled package information includes the exact API hash.
3. During migration, a package without an ARH hash is checked in APIView.
4. The pipeline fails when approval is missing or cannot be checked.
5. `Skip.CheckPackageApproval` is the authorized emergency override.
6. After publishing, `azsdk package mark-released` records the release.

APIView is removed after all languages are onboarded to ARH. If ARH is unavailable
afterward, release uses the authorized break-glass override.

The target ARH result combines required SDK API and package-name approval.

Questions answered by the release design:

- **Question:** Which system enforces SDK API approval for release?

  **Answer:** The language release pipeline calls the shared
  `azsdk package get-approval-status` command defined by #16660.

- **Question:** What system computes the API hash?

  **Answer:** Each language repository owns API artifact generation and defines the
  hash for that language.

- **Question:** Does the release pipeline read both APIView and ARH during
  migration?

  **Answer:** Yes.

- **Question:** What ends the APIView and ARH migration period?

  **Answer:** All languages are onboarded to ARH, then APIView is retired.

- **Question:** What happens when ARH is unavailable after APIView is retired?

  **Answer:** An authorized break-glass override can unblock the release.

Questions not answered by the reviewed comments:

- [ ] Is approval required before the working PR merges, before release execution,
  or both?
- [ ] Which release types require architecture review: first preview, preview
  update, first GA, GA update, and patch?

---

## Scope boundary

The process has three separate review groups:

1. Breaking Change Board
2. Stewardship Board
3. SDK Architecture Board

This work replaces the SDK Architecture Board's APIView and SDK review issue flow.
It stores package-name decisions in ARH but does not move the package-name approval
action away from the GitHub spec PR.

| ARH covers | Separate process |
|------------|------------------|
| SDK public API review | Data-plane specification review by the Stewardship Board |
| SDK API review PR | Breaking-change approval |
| SDK architect assignment and decision | ARM and other specification review |
| API approval for the exact API hash | General SDK code review |
| Package-name approval record | Package-name approval action on the spec PR |
| SDK Architecture Board dashboard | Other release approvals |

This boundary is settled. Stewardship and Breaking Change review remain separate.
