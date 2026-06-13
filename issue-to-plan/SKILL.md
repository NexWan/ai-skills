---
name: issue-to-plan
description: >-
  Turn a GitHub issue into a scoped, reviewed implementation plan on a
  ready-to-work branch. Use this whenever the user wants to start, pick up,
  scope, investigate, groom, or plan work for a GitHub issue — for example
  "let's work on issue #42", "pick up the AI-title-generation issue", "plan
  out github.com/org/repo/issues/12", or when they paste an issue title,
  number, or URL and want to begin addressing it. The skill reads the issue
  and its comments from the current repository's git remote, explores the
  codebase to determine scope, asks targeted clarifying questions, creates a
  branch off the origin default branch, and presents an implementation plan
  for approval in plan mode — then posts the approved plan back to the issue
  as a comment. It stops at the approved plan and does not write the
  implementation. Trigger even when the user does not say the word "plan", as
  long as they clearly want to begin work on a specific existing GitHub issue.
  Do not use this for filing new issues or for planning work unrelated to a
  GitHub issue.
compatibility: Requires the `gh` CLI (authenticated) and `git`, run from inside a git repository with a GitHub `origin` remote.
---

# Issue → Plan

Take a GitHub issue from "I should look at this" to "I have a clear, approved
plan and a branch ready to code on." You read the issue properly, ground it in
the actual codebase, resolve the unknowns with the user, and produce a plan
they've signed off on.

**Where this skill stops:** at the approved plan. You create the branch and
post the plan to the issue, then hand off. You do **not** write the
implementation — that's a separate, deliberate step the user starts when ready.

Hold this boundary even if the issue looks trivial. A two-line fix still
benefits from a confirmed understanding and a clean branch, and the user chose
plan-first on purpose.

## Before you start

This skill operates on the **current repository's** GitHub remote. Confirm the
environment up front so you fail fast with a clear message instead of halfway
through:

```bash
gh auth status                                   # is gh authenticated?
gh repo view --json nameWithOwner,defaultBranchRef \
  --jq '{repo: .nameWithOwner, default_branch: .defaultBranchRef.name}'
```

If `gh` isn't authenticated, or there's no GitHub remote, say so and stop —
don't guess. Note the `repo` and `default_branch` values; you'll use both.

## Workflow

### 1. Resolve the issue

The user gives you a title, a number (`#7` or `7`), or a URL. Turn it into a
single, confirmed issue:

- **URL** (`https://github.com/OWNER/REPO/issues/N`) — parse `OWNER/REPO` and
  `N`. If that repo differs from the current `nameWithOwner`, **stop and flag
  it**: the branch you'd create lives in *this* workspace, so a mismatch
  usually means the wrong folder is open. Ask before continuing.
- **Number** — read it directly from the current repo (next step).
- **Title or keywords** — search, since the user rarely types the title
  verbatim:
  ```bash
  gh issue list --search "<keywords>" --state open --limit 10 \
    --json number,title,labels,url
  ```
  One obvious match → use it (and say which one). Several plausible matches →
  show the shortlist and let the user pick. No match → widen the search or
  include `--state all` before giving up.

### 2. Read the issue — all of it

```bash
gh issue view <number> \
  --json number,title,state,author,labels,assignees,milestone,url,body,comments
```

Read the **body and every comment**. Comments are where the real information
usually lives: repro steps, a maintainer narrowing the scope, a decision that
reverses the title, links to related work. Skipping them is how you confidently
plan the wrong thing.

Then play it back to the user in 2–4 sentences: what the issue is asking for,
in your own words. This catches a misread before it costs anything.

### 2.5. Determine the work type

Pick the type — it drives the branch prefix and frames the plan. Use the
labels first, then the content:

| Signal | Type |
| --- | --- |
| label `bug`, "broken/regression/crash", a defect | `fix` |
| label `enhancement`/`feature`, new capability | `feat` |
| restructure with no behavior change | `refactor` |
| docs only | `docs` |
| build/CI/deps/tooling | `chore` |

When labels and content disagree, trust the content and say why.

### 3. Determine scope — ground it in the code

An issue describes a symptom or a wish; the plan has to land in real files.
Explore the codebase enough to answer "what would actually change?" — search
for the relevant modules, entry points, and call sites. Lean on subagents
(e.g. an `Explore` agent) for breadth when the surface is large.

You're trying to establish:

- **Where** the change lives — the specific files/functions/components.
- **In scope vs. out of scope** — name what you're deliberately *not* touching.
  Explicit boundaries are what keep a plan honest.
- **Unknowns and risks** — anything that could turn a small change large, or
  that you can't determine from the code alone (these become your questions).

### 4. Ask targeted clarifying questions

This is the highest-leverage step — a plan is only as good as the
understanding behind it. Ask about the gaps you actually found in steps 2–3,
not a generic checklist.

Good questions are **specific and grounded**: "The issue says 'generate a
title' — should that fire once after the first message, or update as the
conversation grows?" beats "Any other requirements?" Pull from: acceptance
criteria, scope boundaries, preferred approach when several exist, affected
surfaces (API/UI/data/config), backward-compatibility, and whether tests are
expected.

Use the `AskUserQuestion` tool for discrete choices (it's quick for the user);
ask open ones in plain text. **Batch** them — don't trickle one at a time.
**Skip** anything the issue or code already answers; asking what you could have
read yourself wastes the user's attention. If after reading everything there's
genuinely nothing ambiguous, say so and move on rather than manufacturing
questions.

### 5. Propose the branch and present the plan (plan mode)

Compose the plan using the template below, then present it for approval with
**plan mode** (`ExitPlanMode`). Include the **proposed branch name** in the
plan so the user approves it along with everything else — they can rename it on
the spot.

Branch name: `<type>/issue-<number>-<slug>`, where `<slug>` is a short kebab
slug of the title (~3–5 meaningful words, drop filler). Example:
`feat/issue-7-ai-title-generation`. Mirror the repo's existing convention if it
clearly differs — match what you see in `git branch` / recent history.

If the user revises, fold in the feedback and re-present. Iterate until they
approve. (If `ExitPlanMode` isn't available in this environment, present the
same plan inline and ask for an explicit "go ahead.")

**Why the branch waits until approval:** nothing in the repo or on GitHub
changes until the user signs off. The plan stays a safe, read-only proposal,
and a single approval covers the branch, the plan, and the issue comment
together.

### 6. On approval — create the branch, post the plan, hand off

Now, and only now, make the changes:

1. **Check the working tree is clean** before switching:
   ```bash
   git status --porcelain
   ```
   If there are uncommitted changes, **don't silently stash or discard them** —
   surface them and let the user choose (stash, commit first, or abort). You
   don't know what that work is.

2. **Create the branch off the up-to-date default branch:**
   ```bash
   git fetch origin
   git switch --create <branch> --no-track origin/<default_branch>
   ```
   `--no-track` keeps the new branch's upstream from defaulting to the base;
   the user's later `git push -u` will set the right one.

3. **Post the approved plan as a comment** (write it to a temp file first to
   keep multi-line markdown intact):
   ```bash
   gh issue comment <number> --body-file <path-to-plan.md>
   ```

4. **Hand off.** Confirm concisely: branch `X` is checked out off
   `origin/<default>`, the plan is posted to issue #N (link it), and
   implementation is the next step whenever they're ready. Do **not** start
   coding.

## Plan template

Use this structure for both the plan-mode presentation and the issue comment.
Keep it concrete — every section should reflect *this* issue and *this*
codebase, not boilerplate.

```markdown
# Plan — <issue title> (#<number>)

## Problem
<2–4 sentences in your own words, grounded in the issue + its comments>

## Scope
- **In scope:** <bullets>
- **Out of scope:** <what you are deliberately not doing>

## Approach
<the chosen approach and the key reason for it; note any alternative you
rejected and why, if it's a real fork in the road>

## Changes
- `path/to/file` — <what changes and why>
- ...

## Steps
1. <ordered, concrete steps an implementer can follow>
2. ...

## Testing & verification
- <how the change will be proven: tests to add/run, manual checks, edge cases>

## Risks & open questions
- <anything that could bite; decisions still pending>

## Branch
`<type>/issue-<number>-<slug>` off `origin/<default_branch>`
```

When posting as the issue comment, add a short footer so readers know its
origin and status, e.g.:
`_Plan drafted with Claude Code; subject to change during implementation._`

## What this skill does not do

- **Implement.** It stops at the approved plan and the ready branch.
- **File or edit issues.** It reads an existing issue and comments a plan; it
  doesn't create new issues or rewrite the original.
- **Push or open a PR.** The branch is local. Pushing and PRs are later steps.
- **Touch GitHub before approval.** The only write is the post-approval comment.
