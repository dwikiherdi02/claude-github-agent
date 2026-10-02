# Issue → Branch → PR Workflow

This file governs the agent's ability to implement a fix/feature for a GitHub issue on a
dedicated branch and open a pull request from it. Instructional text here is in English;
all actual GitHub-facing output (branch descriptions in PR body, commit messages, PR title/body,
issue comments) must be written in Bahasa Indonesia per the Language Policy in `CLAUDE.md`.

---

## Reality Check: Available Tools

The connected GitHub MCP server does **not** expose `create_or_update_file` or `push_files`.
The only write-capable tools available for this workflow are:

- `create_branch` — creates a new ref on the remote, copied from a base branch
- `create_pull_request` — opens a PR from an existing head branch into a base branch
- `update_pull_request` — edits title/body/draft-state/reviewers of a PR already created
- `add_issue_comment` — optional, to link the resulting PR back on the issue

There is **no** remote file-write tool. Actual code changes must be made with local git
(via the `Bash` tool) against a checked-out copy of the repository, committed, and pushed
with `git push`. Only after real commits exist on the branch should a PR be opened — a PR
with no diff against its base is not useful and must not be created.

If no local clone of the target repository is available, say so explicitly and ask the user
how they want the code changes delivered (e.g. clone it first) rather than opening an empty
branch/PR.

---

## Branch Naming Convention

```
<type>/issue-<number>-<slug>
```

- `type` ∈ `fix`, `feature`, `perf`, `security`, `chore` — match the issue's label/category
- `number` — the GitHub issue number
- `slug` — short kebab-case summary of the issue title (max ~5 words, ASCII only)

Example: issue #123 "Login gagal saat email mengandung karakter unicode" → `fix/issue-123-unicode-email-login`

---

## Mandatory Steps

1. **Read the issue fully** (`issue_read`). Understand the acceptance criteria / reproduction
   steps / expected behavior. If the scope is ambiguous or the issue spans multiple unrelated
   changes, ask the user before implementing anything — do not guess scope.
2. **Search codebase conventions** before writing any code — apply Rule 3 (Codebase Architecture
   Adherence) and Rule 4 (Behavioral Preservation) from `CLAUDE.md`. This is a fix/feature, not a
   refactor: do not touch code outside what the issue requires.
3. **Confirm the target repository and base branch** (defaults to the repo's default branch
   unless the issue or user says otherwise).
4. **Confirm the plan with the user** before creating the branch or writing code — pushing
   commits and opening a PR are actions visible to others, so get explicit agreement on the
   approach first.
5. **Create the branch** with `create_branch`, using the naming convention above, from the
   confirmed base branch.
6. **Implement the change locally** (Bash + git) against a clone of the repository, following
   existing patterns found in step 2. Do not introduce unrelated refactors, renames, or style
   changes to working code.
7. **Commit and push**:
   - Commit message(s) in Bahasa Indonesia, referencing the issue, e.g. `Perbaiki #123: <ringkasan>`
   - Push only to the newly created feature branch — never to `main`, `master`, `develop`, or
     any protected/release branch
   - Never force-push
8. **Open the pull request** with `create_pull_request`:
   - `head` = the feature branch just pushed; `base` = the confirmed base branch
   - Title and body in Bahasa Indonesia
   - Use the repository's PR template if one exists (`pull_request_template.md` or
     `.github/PULL_REQUEST_TEMPLATE/`), otherwise use the default structure below
   - Body must include `Closes #<number>` (or `Menutup #<number>`) to auto-link the issue
   - Open as **draft** by default unless the user explicitly asks for a ready-for-review PR —
     this keeps a human in control of requesting reviewers/merging
9. **Optionally comment on the issue** (`add_issue_comment`) linking the new PR, only if the
   user asked for this or it is the team's convention.
10. **Report back**: branch name, PR URL, PR number, and a one-line summary in Bahasa Indonesia.

---

## Default PR Body Template (when no repo template exists)

```markdown
## Konteks

[Ringkasan singkat masalah dari issue, dan pendekatan yang diambil]

## Perubahan

- [Perubahan pertama]
- [Perubahan kedua]

## Cara Pengujian

[Langkah untuk memverifikasi perubahan ini bekerja sesuai harapan]

Closes #<number>
```

---

## Safety Rules

- Never push commits directly to `main`, `master`, `develop`, `production`, or any release
  branch — always work on the newly created feature branch.
- Never force-push, under any circumstance in this workflow.
- Never delete a branch unless the user explicitly asks (e.g. after the PR is merged/closed).
- Never merge, approve, or request-changes on the PR this workflow creates — that remains the
  human reviewer's decision (see `scope-limits.md`).
- One issue → one branch → one PR, unless the user explicitly asks for the work to be split
  across multiple PRs.
- If, while implementing, you discover the fix requires changes the issue didn't scope (e.g. a
  dependency bump, a schema change), stop and confirm with the user before proceeding.
- If a new dependency is required to implement the fix, apply the same package security
  evaluation described in `CLAUDE.md` Rule 6 before adding it, and mention the evaluation in the
  PR body.
