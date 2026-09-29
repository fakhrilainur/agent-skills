---
name: glab
description: Uses the glab CLI to create, view, inspect, or operate GitLab resources. Use when a terminal task explicitly calls for glab to work with GitLab merge requests, issues, releases, repositories, packages, or API endpoints.
---

# GitLab CLI (glab)

## Overview

Use `glab` for GitLab work that belongs in the terminal. Keep read-only inspection separate from actions that create, update, merge, retry, cancel, delete, rotate, or publish resources.

## When to Use

- Working with GitLab merge requests, issues, CI/CD pipelines or jobs, releases, projects, or packages.
- Querying or automating a GitLab REST or GraphQL API endpoint with `glab api`.
- Working with GitLab.com, GitLab Dedicated, or a self-managed GitLab instance.

Do not use this skill for local Git history, branching, commits, or conflicts with no GitLab operation; use `git-workflow-and-versioning` instead. When formatting and creating a review-ready GitLab merge request with standardized description templates, touchpoints, and test evidence, follow the `gitlab-mr-create` skill instead. Use `ci-cd-and-automation` when changing pipeline definitions rather than operating an existing pipeline.

## Process

1. Establish the target before running a mutating command. In a Git repository, `glab` normally detects the GitLab host and project from the remotes. For another project, pass `--repo group/project`; for another instance, use the relevant host option or `GITLAB_HOST`. Inspect with `glab auth status` and a read-only command such as `glab repo view` when context is uncertain.

2. Confirm authentication without exposing credentials. Use interactive `glab auth login` when the user has authorized setup. In CI, prefer `GLAB_ENABLE_CI_AUTOLOGIN=true` with the job token for commands that support it. Never print, embed, or pass a token in a command line, commit, issue, merge request, or shell history.

3. Use the highest-level command that represents the requested GitLab object.

   - Merge requests: inspect with `glab mr list`, `glab mr view`, or `glab mr diff`; create with explicit title, target branch, reviewers, labels, and draft status as appropriate (or follow the `gitlab-mr-create` skill for standardized review templates). Treat `glab mr merge` and auto-merge as externally mutating actions requiring confirmation immediately before execution.
   - Issues: use `glab issue list`, `view`, `create`, `update`, and `note`. Prefer `--description-file` for substantial Markdown and verify templates exist locally before passing `--template`.
   - CI/CD: inspect with `glab ci status`, `view`, or `trace`; retry, cancel, run, trigger, or delete only with an identified pipeline or job and authorization. Use `glab ci lint` before relying on changed CI configuration.
   - Releases: inspect the tag and release first. For a new tag, state the intended ref explicitly with `--ref`; `glab release create <tag>` can otherwise create a tag from the default branch. In CI, do not assign `CI_JOB_TOKEN` to `GITLAB_TOKEN`; enable CI auto-login instead.

4. Use `glab api` only when a first-class command does not cover the operation. Read the endpoint documentation, specify `--method` explicitly for mutations, and validate the host, project path, and payload shape before sending it. `glab api` defaults to `GET` without fields but to `POST` when fields are supplied. Use `--paginate` for complete list retrieval; prefer `--output ndjson` for large streams processed by `jq`.

5. Before external mutation, present the exact target and meaningful effects, then obtain any required user confirmation. Prefer a dry or read-only inspection when one exists. After an authorized command, verify the resulting resource with its URL, IID/ID, state, or a follow-up read command.

For command-specific flags and version-sensitive behavior, consult the [GitLab CLI command reference](https://docs.gitlab.com/cli/commands/).

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The current repository must be the right project." | Remotes and authentication can resolve to different GitLab instances; verify or explicitly set the target. |
| "Adding fields to `glab api` is harmless." | It changes the default method to `POST`; name the method for every mutation. |
| "The release tag obviously points at this commit." | A missing tag can be created from the default branch. Use `--ref` when the intended commit matters. |
| "A CI job token works wherever `GITLAB_TOKEN` works." | The Releases API expects the CI job token in a different header; use CI auto-login. |

## Red Flags

- A mutating command is about to run without a project, instance, or resource identifier confirmed.
- A token appears in a command, log, config diff, or generated file.
- `glab api` is used for a routine MR, issue, pipeline, or release action already supported by `glab`.
- A release, merge, retry, cancel, delete, revoke, or token rotation is treated as a read-only lookup.

## Verification

- [ ] The GitLab host and project were inferred safely or explicitly identified.
- [ ] Authentication uses an approved mechanism and no credential was exposed.
- [ ] A first-class `glab` command was chosen when available; API method and payload were explicit otherwise.
- [ ] Any destructive or externally visible action was confirmed immediately before execution.
- [ ] A follow-up command or returned URL/identifier verifies the resulting GitLab state.
