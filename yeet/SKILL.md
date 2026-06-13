---
name: "yeet"
description: "Publish local changes to GitHub end-to-end: confirm scope, branch if needed, stage intentionally, commit, push, and open a draft pull request — preferring the GitHub app bundled with this plugin for PR creation and falling back to `gh` when connector coverage is insufficient. Use whenever the user explicitly wants the full local-to-PR flow in one go: 'publish my changes', 'push this and open a PR', 'ship it', or 'yeet it'. Not for a standalone commit (use a commit skill) or routine git status/diff inspection."
---

# GitHub Publish Changes

## Overview

Use this skill only when the user explicitly wants the full publish flow from the local checkout: branch setup if needed, staging, commit, push, and opening a pull request.

This workflow is hybrid:

- Use local `git` for branch creation, staging, commit, and push.
- Prefer the GitHub app from this plugin for pull request creation after the branch is on the remote.
- Use `gh` as a fallback for current-branch PR discovery, auth checks, or PR creation when the connector path cannot infer the repository or head branch cleanly.

## Prerequisites

- Local `git` repository with a clean understanding of which changes belong in the PR.
- GitHub CLI `gh` is strongly recommended for auth checks and as a fallback for PR creation.
  - Check `gh --version`. If `gh` is missing, the plugin can still perform the core workflow (branch, commit, push) but will not be able to open a PR without user guidance.
  - If PR creation is needed and `gh` is missing, ask the user to install `gh` and authenticate with `gh auth login` before continuing.

## Naming conventions

Pick one change type that best fits the diff and use it consistently across the branch, commit, and PR title. Common types: `feat` (new capability), `fix` (bug fix), `refactor` (behavior-preserving cleanup), `docs`, `chore` (deps/config/tooling), or `improvement` (an enhancement that isn't a clear `feat`).

- Branch: `{type}/{short-kebab-description}` — only when starting from `main`, `master`, or the default branch.
- Commit: `{type}: {terse description}` — conventional-commit style keeps history scannable.
- PR title: `{type}: {description}` — summarize the whole diff, not just the latest commit.

## Workflow

1. Confirm intended scope.
   - Run `git status -sb` and inspect the diff before staging.
   - If the working tree contains unrelated changes, do not default to `git add -A`. With a human in the loop, ask which files belong in the PR. Running autonomously, infer scope from the request, stage only those files by explicit path, and state what you included and excluded — the goal is to never sweep in unrelated work silently, not to halt.
2. Determine the branch strategy.
   - If on `main`, `master`, or another default branch, create `{type}/{short-kebab-description}` (see Naming conventions).
   - Otherwise stay on the current branch.
3. Stage only the intended changes.
   - Prefer explicit file paths when the worktree is mixed.
   - Use `git add -A` only when the user has confirmed the whole worktree belongs in scope.
4. Commit with the chosen change type and a terse description (see Naming conventions) — e.g. `git commit -m "fix: handle empty diff"`.
5. Run the checks that matter for this diff before pushing, if they haven't run already.
   - Discover them from the project instead of guessing: `package.json` scripts (`test`, `lint`, `typecheck`, `build`), a `Makefile`, a `pre-commit` config, or the workflows under `.github/workflows`.
   - When the change is a fix, reproduce the failing check first so the PR can cite a concrete before/after.
   - Run only what's relevant to what changed — a docs-only edit doesn't need the full test suite.
   - If a check fails because a tool or dependency is missing, install it and rerun once. If it still fails, surface the failure instead of pushing past it.
   - Don't block the push on pre-existing failures unrelated to this change; call them out in the summary instead.
6. Push with tracking: `git push -u origin $(git branch --show-current)`.
7. Open a draft PR.
   - Both PR paths below need a remote that resolves to GitHub. If none does (for example `gh repo view` reports no GitHub host), don't force it: the branch is already pushed, so report that the PR can't be opened without a GitHub remote and stop, per Write Safety.
   - Prefer the GitHub app from this plugin for PR creation after the push succeeds.
   - Derive `repository_full_name` from the remote, for example by normalizing `git remote get-url origin` or by using `gh repo view --json nameWithOwner`.
   - Derive `head_branch` from `git branch --show-current`.
   - Derive `base_branch` from the user request when specified; otherwise use the remote default branch, for example via `gh repo view --json defaultBranchRef`.
   - If the branch is being pushed from a fork or the PR target differs from the remote that was just pushed, prefer `gh pr create` fallback because the connector PR creation flow expects one repository target and may not encode cross-repo head semantics cleanly.
   - If connector-based PR creation cannot infer the repository or branch cleanly, fall back to `gh pr create --draft --fill --head $(git branch --show-current)`.
   - Write the PR body to a temp file with real newlines when using CLI fallback so the markdown renders cleanly.
8. Summarize the result with branch name, commit, PR target, validation, and anything the user still needs to confirm.

## Write Safety

- Never stage unrelated user changes silently.
- Never push a mixed worktree without first establishing scope — confirmed by the user, or (when autonomous) inferred from the request and stated explicitly (see step 1).
- Default to a draft PR unless the user explicitly asks for a ready-for-review PR.
- If the repository does not appear to be connected to an accessible GitHub remote, stop and explain the blocker before making assumptions.

## PR Body Expectations

The PR description should use real Markdown prose and cover the points that apply (a feature PR doesn't need a root-cause line):

- what changed
- why it changed
- the user or developer impact
- the root cause, when the PR is a fix
- the checks used to validate it
