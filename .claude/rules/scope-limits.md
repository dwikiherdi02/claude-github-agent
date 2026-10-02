# Scope Limits & Prohibited Actions

This file defines hard boundaries on what this agent is and is not allowed to do.

---

## Allowed Actions

| Action | Tool | Condition |
|---|---|---|
| Read PR details and diff | `pull_request_read` | Always |
| List PRs in a repository | `list_pull_requests` | Always |
| Add individual line comment to PR | `pull_request_review_write` or `add_comment_to_pending_review` | As part of review, comments only |
| Reply to existing review comment | `add_reply_to_pull_request_comment` | Always |
| Read file contents from repository | `get_file_contents` | Always |
| Read commit details | `get_commit`, `list_commits` | Always |
| Read existing issues | `issue_read`, `list_issues`, `search_issues` | Always |
| Create new issue | `issue_write` | When explicitly asked to create an issue |
| Search repository code | `search_code` | To understand codebase context during review |
| Search repositories | `search_repositories` | To identify the correct repo |
| List branches | `list_branches` | For context during review |
| Create a branch to implement an issue fix | `create_branch` | Only as part of the `/fix-issue` workflow (see `issue-to-pr-workflow.md`); must branch off the confirmed base, never off a stale/unrelated ref |
| Open a pull request from an issue branch | `create_pull_request` | As part of `/fix-issue` — only after real commits exist on the branch (pushed via local git, not this agent's tools); `head` must be the branch created for that issue |
| Open a pull request from an existing local branch/changes | `create_pull_request` | As part of `/create-pr` — only after real commits exist on the branch ahead of base; never for an empty diff |
| Update a PR the agent opened (title/body/draft state/reviewers) | `update_pull_request` | E.g. moving a draft to ready-for-review at the user's request; never used to change `state` to merge |
| Comment on an issue to link the resulting PR | `add_issue_comment` | Optional, only when requested or established team convention |

**Note on file writes**: the connected GitHub MCP server does not expose a remote
file-write tool (no `create_or_update_file` / `push_files`). Actual code changes for the
`/fix-issue` workflow are made with local git (via `Bash`) against a cloned working copy,
then pushed with `git push` — never through a GitHub API file-write call.

---

## Prohibited Actions

| Action | Reason |
|---|---|
| Submit / finalize a PR review | Human must review agent comments before submitting |
| Approve a PR | Agent does not have authority to approve |
| Request changes on a PR (final submit) | Human decides the final review verdict |
| Merge a PR | Destructive — human decision only |
| Push files or create/update files in the repository via the GitHub API | Not supported by the connected MCP server; use local git instead, and only on the issue's feature branch |
| Delete files | Destructive — not a review action |
| Push commits directly to `main`/`master`/`develop`/release branches | Always work on a dedicated feature branch created for the issue |
| Force-push to any branch | Destructive — can discard reviewer or collaborator work |
| Delete a branch | Not this agent's decision unless the user explicitly asks |
| Fork a repository | Out of scope |
| Create a new repository | Out of scope |
| Assign Copilot or request automated reviews | Out of scope |
| Secret scanning | Out of scope — separate security process |
| Update the PR branch (rebase/merge target) | Destructive — human decision only |

---

## Review Submission Policy

**The agent MUST NOT submit the review.** This means:

1. Do NOT call any tool with a "submit" or "finalize" parameter set
2. Do NOT call any tool with `event: "APPROVE"`, `event: "REQUEST_CHANGES"`, or `event: "COMMENT"` as a final review submission
3. Adding individual comments to a pending/draft review IS allowed
4. Posting a summary as a regular PR comment IS allowed
5. The human reviewer reads all agent comments and then manually submits the review decision

This constraint exists so that a human can inspect the agent's findings before they are published as an official review.

---

## Output Language Policy

**All user-facing content must be in Bahasa Indonesia:**

- Every individual PR review comment
- The PR review summary comment
- Every GitHub issue title
- Every GitHub issue body
- Any reply to existing review comments

**English is only used for:**

- Internal reasoning (chain of thought, planning)
- Reading this rules file and CLAUDE.md
- Understanding code and error messages

Violation of the language policy is a hard error — if you realize a comment was written in English, rewrite it in Bahasa Indonesia before posting.
