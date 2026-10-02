Create a pull request from the following branch/changes: $ARGUMENTS

Execute every step below in order. This command is for opening a PR from work that already
exists locally (a branch you or the user already implemented), as opposed to `/fix-issue`
which implements the change *and* opens the PR end-to-end. Follow
`.claude/rules/issue-to-pr-workflow.md` and `.claude/rules/scope-limits.md` for the
underlying rules — this command is the step-by-step checklist for the "just open the PR"
case.

---

## Step 1: Identify the Source and Target

Determine from the input (or ask if not specified):
- Target repository
- Head branch (the branch containing the changes to be opened as a PR)
- Base branch (defaults to the repository's default branch unless stated otherwise)
- Whether this PR relates to an existing issue (for `Closes #<number>`)

---

## Step 2: Verify Real Commits Exist

Before doing anything else, confirm the head branch actually has commits ahead of the base
branch:
1. Use `Bash`/`git` (if a local clone is available) or `list_commits` / `get_commit` via the
   GitHub MCP tools to compare head vs. base
2. **Never open a PR with no diff against its base** — if the head branch has no new commits,
   stop and tell the user rather than creating an empty PR

If the branch doesn't exist on the remote yet but exists locally, push it first (never
force-push, never push to `main`/`master`/`develop`/release branches as the *base* of a push).

---

## Step 3: Review the Diff Before Writing the Description

Read the actual changes (`git diff <base>...<head>` locally, or diff via the GitHub tools) so
the PR description accurately reflects what changed — do not describe changes you haven't
actually looked at.

---

## Step 4: Confirm the Plan

Before opening anything, summarize for the user:
- Repository, head branch, base branch
- A one-line description of what the PR will contain
- Whether it will be a draft (default) or ready-for-review

Get explicit confirmation — opening a PR is visible to others/collaborators.

---

## Step 5: Compose the PR in Bahasa Indonesia

Search the repository for a PR template (`pull_request_template.md` or
`.github/PULL_REQUEST_TEMPLATE/`) and use it if found. Otherwise use this default:

```markdown
## Konteks

[Ringkasan singkat latar belakang perubahan ini]

## Perubahan

- [Perubahan pertama]
- [Perubahan kedua]

## Cara Pengujian

[Langkah untuk memverifikasi perubahan ini bekerja sesuai harapan]

Closes #<number>
```

Omit the `Closes #<number>` line entirely if this PR is not tied to an issue.

If a new dependency was introduced in the diff, evaluate it per Rule 6 in `CLAUDE.md` and
include the `Rekomendasi Package` evaluation in the PR body.

---

## Step 6: Open the Pull Request

Call `create_pull_request`:
- `head` = the confirmed head branch, `base` = the confirmed base branch
- Title and body in **Bahasa Indonesia**
- Create as **draft** by default unless the user explicitly asked for a ready-for-review PR

---

## Step 7: Report Back

Report to the user, in Bahasa Indonesia:
- PR URL and number
- One-line summary of what the PR contains

---

## Constraints

- Never open a PR with no real commits behind it
- Never push directly to `main`/`master`/`develop`/release branches, and never force-push
- Never merge, approve, or request-changes on the PR — that remains a human decision
- Title and body must be in **Bahasa Indonesia**
- If the diff includes a new dependency, apply the package security evaluation from Rule 6
  (`CLAUDE.md`) before describing it as safe
