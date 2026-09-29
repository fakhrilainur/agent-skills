---
name: gitlab-mr-create
description: Formats and creates standardized GitLab merge requests using glab CLI. Use when user asks to create, format, publish, or prepare a GitLab MR with review-ready templates, validation matrices, test evidence, review focus, and developer pre-flight checklists.
---

# GitLab MR Creation (gitlab-mr-create)

## Overview

Creates review-ready GitLab Merge Requests using the `glab` CLI with standardized descriptions derived from specs, code touchpoints, and deterministic test evidence.

## When to Use

- Creating, opening, drafting, or publishing a review-ready GitLab Merge Request from the current branch.
- Standardizing MR descriptions using specs and touchpoints (see `spec-driven-development`).
- Embedding deterministic test evidence from unit tests (`test-driven-development`) or runtime daemon verification (`live-smoke-testing`).
- Pre-flight checking an MR before handing off to human/lead review.

## When NOT to Use

- Local git operations, branching, or atomic commits without remote GitLab interaction (follow the `git-workflow-and-versioning` skill).
- Operating non-MR GitLab resources such as pipelines, issues, or releases (see the `glab` skill).
- Required test evidence or validation scenarios are missing or failing.
- Working with GitHub repositories (use GitHub PR workflow instead).
- Creating MR directly from default branches (`main`, `master`, `develop`) without explicit override.

## Process

### 1. Inspect Branch and Remote State
Use safe read-only commands first:
- Detect current branch: `git branch --show-current`. Stop if on `main`/`master`/`develop` unless explicitly approved.
- Detect target branch: default repository branch via `glab repo view` or remote tracking branch.
- Inspect commits and touched files: `git log --oneline <target>..HEAD` and `git diff --stat <target>..HEAD`.

### 2. Detect User and Resolve Reviewer
- Run `glab api user` to read the `username` field as default assignee. Fall back to asking user if auth is missing.
- Prompt for lead reviewer: `Who is your lead to assign as reviewer? (type username, or 'skip' to leave empty)`.

### 3. Draft Standardized Description
Check for linked specs (`docs/specs/<feature-slug>.spec.md`) and select mode:
- **Lane A (Lean / Standard):** Fill Summary, Context & Lane, Changes, Test Evidence, Review Focus, and Developer Checklist. (Mark "N/A" for Production Readiness & Rollout).
- **Lane B (Full / Heavy):** All sections are mandatory, including Validation Scenarios Matrix, Database/Schema, and Production Readiness & Rollout.

> **Security Rule:** Never paste raw secrets, bearer tokens, or database passwords in the MR description, CLI arguments, or test output evidence. Redact all tokens (`[REDACTED]`).

### 4. Preview Format & Confirmation Gate
Before creating the MR, output this exact preview:

````markdown
## MR Preview

Source branch: `<source>`
Target branch: `<target>`
Draft: Yes / No
Labels: `<labels or none>`
Reviewers: `<reviewers or none>`
Assignee: `<assignee or none>`

> Reminder: confirm the reviewer above is your lead. Add or change it now if needed.

### Title
<type>(<scope>): <concise imperative title>

### Description
<Standardized MR Description Markdown>

### Command
```bash
glab mr create --source-branch "<source>" --target-branch "<target>" --title "<title>" --description-file /tmp/mr_desc.md
```

Create this GitLab MR now? (y/n)
````

**Do not proceed unless the user clearly confirms (`y` / `yes`).**

### 5. Execution & Verification
1. Write the finalized MR markdown description to a temporary file (e.g. `/tmp/mr_description.md` or `.git/mr_description.md`). Never interpolate multiline markdown containing backticks or shell symbols directly into inline `-d` string arguments.
2. Run `glab mr create` with explicit `--source-branch`, `--target-branch`, `--title`, and `--description-file <file-path>`.
3. Output resulting MR URL, IID, and status.
4. Delete the temporary description file after the command succeeds or fails.

## Standardized MR Description Template

````markdown
## Summary
- <Main behavior change in one imperative line>
- <Supporting change in one line (omit if only one change)>

## Context & Lane
- **Lane:** Lane A (Standard) | Lane B (Heavy)
- **Risk Level:** Low | Medium | High
- **Verifiability:** Cheap (unit/deterministic) | Expensive (manual/staging/sandbox)
- **Linked Spec:** `docs/specs/<feature-slug>.spec.md` (or "N/A - Direct fix/refactor")
- **Why:** <Problem this solves or user impact created. 1-3 sentences referencing linked spec>

## Codebase Touchpoints & Changes
- **Core Code Changes:**
  - `MODIFY/CREATE` `<path:line>`: <Summary of changed symbols/functions>
- **Contract / Schema Changes:** None | <Brief description of DDL / API changes>
- **Test Changes:** <New tests, modified tests, table-driven test matrix>

## Test Evidence
```text
<command>: exit code 0, PASS (<timing/coverage>)
<command lint/build>: 0 errors, PASS
```

## Validation Scenarios Matrix (Proof of Done)
> Required for all feature additions, bug fixes, and behavioral changes. (Can mark N/A for pure refactors).

| VS ID | Linked AC | Type | Actor | Environment | Command / Reproduction | Result | Evidence |
|---|---|---|---|---|---|---|---|
| VS-1 | AC-1 | Unit | AI | Local | `npm test -- tests/... -t "TC-1"` | PASS | Exit code 0 |
| VS-2 | AC-2 | Integration | AI | Local/Testcontainers | `npm run test:integration` | PASS | Exit code 0 |

## Review Focus
- <Hotspot files or complex logic algorithms requiring reviewer's special attention>
- <Key edge-cases, assumptions, or tradeoffs made>

## Production Readiness & Rollout (Mandatory Lane B; "N/A" for Lane A)
- **Production Impact:** Yes | No
- **Migration Needed:** No | Yes (`<path to migration SQL>`)
- **Config / .env Keys:** None | `<KEY_NAME>` (documented in `.env.example`)
- **Deployment Sequence:** <1. Config/Secret -> 2. Migration -> 3. App Deploy -> 4. Post-verify>
- **Rollback Plan:** `git revert <commit>` OR <Operational rollback steps without data loss>
- **Monitoring & Alerts:** <Log event names, Prometheus metrics, alert thresholds>

## Developer Pre-Flight Checklist
- [ ] Codebase touchpoints inspected and strictly within approved scope
- [ ] All unit, integration, and contract tests pass deterministically
- [ ] Linter, type check, and static analysis pass with 0 errors
- [ ] Concurrency and race conditions addressed (transactions, idempotency, unique constraints)
- [ ] Zero hard-coded secrets, tokens, or private credentials in code or MR description
- [ ] Documentation / spec updated if behavior or contracts changed
- [ ] Self-review of git diff completed before opening MR

## Notes & Breaking Changes
- **Breaking Changes:** None | <Describe migration path for consumers>
- **Related Issues / MRs:** Closes #<issue-id>
````

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The commit messages are clear, no need for detailed MR description." | Reviewers review the MR description first to evaluate blast radius, risk, and rollback strategy. |
| "I'll run the tests after opening the MR." | Test evidence and validation matrix must be verified locally before opening the MR. |
| "Skip assignee/reviewer prompts to be faster." | Explicit reviewer assignment guarantees review ownership and audit trails. |

## Red Flags

- Executing `glab mr create` without displaying the preview and waiting for explicit user confirmation.
- Submitting an MR with unfilled placeholder text (`<command>`, `<VS-ID>`, `TBD`).
- Raw authorization headers, tokens, or environment passwords pasted into the description or test output.
- Creating an MR from `main`, `master`, or `develop` without explicit confirmation.

## Verification

Before finishing, confirm:

- [ ] Target branch and source branch are verified.
- [ ] Description follows the standardized format (including Lane, Touchpoints, Review Focus, and Checklist).
- [ ] Deterministic test commands and pass/fail evidence are included.
- [ ] Secrets and tokens are completely redacted.
- [ ] User provided explicit confirmation after viewing the preview.
- [ ] Resulting GitLab MR URL is outputted and accessible.
