---
name: create-ticket
description: Drafts, refines, creates, or updates issue and ticket descriptions as human-readable, AI-executable implementation contracts using glab or gh CLI. Use when creating or refining an issue, ticket, or work-item with verifiable requirements, explicit boundaries, test evidence rules, and a Definition of Done. Preview and obtain approval before publishing.
---

# Create Ticket (create-ticket)

## Overview

Write an implementation contract, not a status report. Create, refine, or update issues and tickets—supporting GitLab issues via `glab`, GitHub issues via `gh`, or file-based ticket specifications—with concise Markdown, stable requirement IDs, explicit behavior, and concrete verification criteria so humans can review and AI agents can implement without guessing.

## When to Use

- Drafting, refining, creating, or updating an issue, ticket, or work-item with verifiable requirements and acceptance criteria.
- Establishing an implementation contract as the single source of truth between leads, product owners, developers, and AI agents.
- Creating or updating GitLab issues and work items using the `glab` CLI (see the `glab` skill).
- Creating or updating GitHub issues using the `gh` CLI.
- Converting user stories, bug reports, or feature briefs into AI-executable specifications with traceable requirement IDs (`PREFIX-01`, `PREFIX-02`).

## When NOT to Use

- Designing system-wide architecture or multi-module capabilities before scoping individual tickets; follow the `spec-driven-development` skill instead.
- Breaking down an approved specification into an implementation plan; follow the `planning-and-task-breakdown` skill instead.
- Formatting and publishing a review-ready GitLab Merge Request from a branch; follow the `gitlab-mr-create` skill instead.
- Performing general terminal GitLab CLI operations like pipeline management or releases; follow the `glab` skill instead.
- Documenting architectural decision records or long-lived technical decisions; follow the `documentation-and-adrs` skill instead.

## Process

### 1. Inspect Target and Context

1. **Identify Platform & Target:** Determine the ticket system (GitLab via `glab`, GitHub via `gh`, or Markdown file under `docs/tickets/`). Detect remote host, repository, and target project.
2. **Inspect Existing Tickets:**
   - GitLab: `glab issue view <iid> --repo <group/project> --output json`
   - GitHub: `gh issue view <number> --json title,body,labels,assignees`
   - Preserve relevant user requirements and existing context; do not overwrite unrelated edits.
3. **Trace Constraints:** Inspect relevant repository code, interfaces, and dependencies to ground assumptions in verified code contracts.
4. **Clarify Scope:** Ask questions only when they materially affect boundaries, data contracts, or correctness. Do not invent arbitrary deadlines, rollout gates, or unrequested features.

### 2. Draft the Implementation Contract

Use the standardized contract structure below. Choose a short, domain-specific prefix (e.g., `AUTH-01`, `RATE-01`). Preserve existing IDs during revisions and append new IDs without renumbering unchanged requirements.

````markdown
## Objective

<Capability and observable outcome; a short data flow if useful.>

This description defines the implementation contract. Implementation and test evidence must reference the requirement IDs below.

## Boundaries

**Included**
- <Required capability or integration boundary.>

**Excluded**
- <Explicitly deferred or unrelated work.>

## Requirements And Verification

### <PREFIX>-01 — <Capability>

**Required behavior**
- <Testable contract: inputs, outputs, limits, and applicable failure behavior.>

**Verification**
- <Concrete evidence proving the behavior and relevant edge cases.>

<Repeat for independently verifiable requirements.>

## Evidence Rules

- <Relevant fixture, reference, reproducibility, and dependency-isolation rules.>
- The implementation MR/PR maps its tests and results to the requirement IDs.

## Definition Of Done

- [ ] Every requirement has passing verification evidence.
- [ ] Relevant tests, build, and configured quality checks pass; unrelated failures are documented.
- [ ] Required contracts and configuration are documented.
- [ ] Existing behavior outside the intended change remains compatible.
- [ ] The implementation MR/PR references this ticket and includes verification results.
- [ ] Review is complete with no unresolved correctness issues.

## References

- <Authoritative references needed to implement or verify the task.>
````

**Drafting Rules:**
- Focus on observable behavior, boundaries, inputs, and outputs rather than internal code paths, unless specific files or algorithms are mandatory constraints.
- Make units, rounding, pagination, limits, and failure behaviors explicit.
- Attach edge cases directly to the requirement they affect.
- Omit status chatter, assignee commentary, current-state bug lists, and premature deployment matrices.

### 3. Preview and Confirmation Gate

1. Present the complete proposed ticket title, target project, and description to the user.
2. State clearly that nothing has been published yet.
3. Incorporate feedback into a revised preview.
4. Never publish or mutate a remote issue without explicit user approval.

### 4. Publish After Approval

1. Write the approved content to a temporary description file to prevent shell escaping and truncation issues:
   - GitLab Create: `glab issue create --repo <group/project> --title "<title>" --description-file <file>`
   - GitLab Update: `glab issue update <iid> --repo <group/project> --description-file <file>`
   - GitHub Create: `gh issue create --title "<title>" --body-file <file>`
   - GitHub Update: `gh issue edit <number> --body-file <file>`
2. Never print, embed, or expose secrets, bearer tokens, or sensitive credentials in the ticket description or CLI arguments.
3. When updating an existing ticket, re-read right before publication to prevent overwriting concurrent human edits.

### 5. Verify Publication

1. Inspect the published ticket to confirm creation/update succeeded:
   - Verify title, issue IID/number, and rendered Markdown formatting.
   - Confirm all Definition of Done checkboxes remain unchecked on creation.
2. Output the public ticket URL and a concise summary of the published contract.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This is a simple bug/feature; a one-sentence issue is enough." | Ambiguous tickets lead to guessing, missed edge cases, and rework. Concrete requirements save time for both developers and AI agents. |
| "I can publish the ticket directly without previewing." | Publishing without preview risks publishing wrong scope, unverified assumptions, or leaking private tokens to issue trackers. |
| "I should specify internal file paths and helper functions in the ticket." | Implementation contracts must define observable behavior and boundaries. Dictating internal wiring restricts better refactoring choices. |
| "I can skip the Definition of Done for internal tickets." | Without a testable Definition of Done, neither humans nor agents can objectively prove when the task is complete. |
| "I'll include the current broken code and bug history in the ticket." | Write what the system *must* do as a forward-looking contract, not a retrospective analysis of current defect history. |

## Red Flags

- Publishing, creating, or modifying a ticket on remote tracking systems without showing a preview and obtaining user approval.
- Requirements lacking verification criteria or using vague phrasing like "works reliably" or "handles edge cases properly."
- Storing or printing raw API tokens, credentials, or secrets in ticket descriptions or commands.
- Renumbering existing requirement IDs during ticket revisions, breaking traceability with in-flight MRs/PRs.
- Conflating ticket specifications with status reports, daily logs, or implementation code dumps.

## Verification

Before completing the ticket creation or update, confirm:

- [ ] The ticket has a clear Objective and explicit Boundaries (Included / Excluded).
- [ ] Every requirement has a stable identifier (`PREFIX-01`) and testable verification criteria.
- [ ] Real edge cases, error conditions, and data contracts are explicit.
- [ ] The ticket includes Evidence Rules and a standard Definition of Done checklist.
- [ ] Full preview was displayed and explicit approval was granted before publication.
- [ ] The published ticket was inspected to confirm URL, issue ID, and accurate description formatting.
