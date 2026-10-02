Implement and open a pull request for the following issue: $ARGUMENTS

Execute every step below in order. Follow `.claude/rules/issue-to-pr-workflow.md` for the
full workflow rules — this command is the step-by-step checklist for it.

---

## Step 1: Identify the Issue and Repository

Determine which issue and repository this refers to. If not fully specified, ask before
proceeding. Fetch the issue with `issue_read` and read it completely: title, body, labels,
comments, and any linked PRs or issues.

---

## Step 2: Understand the Requirement

Identify:
- The problem being solved or feature being requested
- Acceptance criteria / reproduction steps / expected vs actual behavior
- Whether the scope is a single, coherent change

If the issue is ambiguous or bundles multiple unrelated changes, ask the user how to scope
the work before writing any code.

---

## Step 3: Explore Codebase Conventions

Before writing code:
1. Search the codebase for at least 2–3 examples of how similar functionality is structured
   (naming, folder layout, error handling, existing utilities) — per Rule 3 in `CLAUDE.md`
2. If the issue involves modifying an existing function, search for all callers and confirm
   the change won't break them — per Rule 4 (Behavioral Preservation) in `CLAUDE.md`
3. Confirm a local clone of the target repository is available for making the actual code
   changes (the GitHub MCP tools available here cannot write file contents remotely — see
   `.claude/rules/issue-to-pr-workflow.md`). If no local clone exists, say so and ask the user
   how to proceed rather than opening an empty branch/PR.

---

## Step 4: Confirm the Plan

Before creating a branch or writing code, summarize for the user:
- The intended approach
- The target repository and base branch
- The proposed branch name (`<type>/issue-<number>-<slug>`)

Get explicit confirmation before proceeding — this workflow pushes commits and opens a PR,
both visible to others.

---

## Step 5: Create the Branch

Call `create_branch` with the confirmed base branch and the branch name from Step 4.

---

## Step 6: Implement the Change

Using Bash + git against the local clone:
1. Check out the new branch
2. Implement only what the issue requires — no unrelated refactors, renames, or style changes
   to working code (Rule 4, CLAUDE.md)
3. Run the project's existing tests/build if available to sanity-check the change

---

## Step 7: Commit and Push

1. Write commit message(s) in Bahasa Indonesia referencing the issue number, e.g.
   `Perbaiki #123: <ringkasan singkat>`
2. Push only to the feature branch created in Step 5
3. Never push to `main`/`master`/`develop`/release branches, and never force-push

---

## Step 8: Open the Pull Request

Call `create_pull_request`:
- `head` = feature branch, `base` = confirmed base branch
- Search the repo for a PR template (`pull_request_template.md` or
  `.github/PULL_REQUEST_TEMPLATE/`) and use it if found; otherwise use the default template in
  `.claude/rules/issue-to-pr-workflow.md`
- Title and body in **Bahasa Indonesia**
- Body must include `Closes #<number>`
- Create as **draft** unless the user explicitly asked for a ready-for-review PR

---

## Step 9: Link Back to the Issue (Optional)

Only if requested or established as team convention, add a comment on the issue
(`add_issue_comment`) linking the new PR, in Bahasa Indonesia.

---

## Step 10: Report Back

Report to the user, in Bahasa Indonesia:
- Branch name created
- PR URL and number
- One-line summary of what was implemented

---

## Constraints

- All GitHub-facing content (commit messages, PR title/body, issue comments) must be in
  **Bahasa Indonesia**
- Never merge, approve, or request-changes on the PR
- Never force-push or push directly to a protected branch
- Never delete branches unless explicitly asked
- Never expand scope beyond what the issue describes without confirming with the user first
- If a new dependency is needed, evaluate it per Rule 6 in `CLAUDE.md` before adding it
