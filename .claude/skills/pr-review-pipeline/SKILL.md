---
name: pr-review-pipeline
description: Run the 4-agent PR review pipeline (pr-data-collector clones the PR branch into temp/ → pr-code-analyzer analyzes the clone → pr-feedback-writer → pr-comment-poster) sequentially on a pull request, save the result as a PENDING review, then delete the clone. Use when the user asks to run the review agents, review a PR with agents, or gives a PR link/number to review.
---

Same pipeline as `/review-pr`; this skill is the natural-language entry point. Rule set: `CLAUDE.md` and `.claude/rules/`.

## Input

A PR reference from the request: URL, `owner/repo#N`, or bare number. Normalize it to `owner/repo#N` and pass that form to every stage.

- Bare number and the repo cannot be inferred from the conversation or the working directory → ask the user once, then stop.
- No PR reference at all → ask once.
- Never guess the repo.

## Pre-flight (before Stage 1, one `pull_request_read` call)

- PR closed or merged → tell the user and ask whether to continue. Do not review silently.
- Draft PR → continue; mention it in the final report.
- The user already has a pending review on this PR → GitHub allows only one, so Stage 4 would fail. Tell the user now and ask whether to continue (findings are reported in chat only) or stop. Never delete or submit their review.

## Pipeline

Run the stages **in order**, one Agent call each (`subagent_type` = agent name). Never skip, merge, or reorder them. Each stage receives its predecessor's output **verbatim**: no summarizing, no re-fetching what a stage already returned.

| # | Agent | Input | Output |
|---|---|---|---|
| 1 | `pr-data-collector` | PR reference | Clone of the **PR head branch** in `temp/pr-<repo>-<N>-<timestamp>/` + clone path, metadata, linked issue (or "Tidak ada issue tertaut"), diff vs `origin/<base>`, existing comments, commits, test-suite presence |
| 2 | `pr-code-analyzer` | Stage 1 output, unchanged (incl. clone path) | Deep analysis inside the clone — changed code per diff + PR description/issue, and every cross-file function/service/helper it calls. Verification Trace + verified findings (or an explicit no-findings result) |
| 3 | `pr-feedback-writer` | **Findings only** from Stage 2. The Verification Trace is internal and must never reach the PR | Final comments per `.claude/rules/comment-format.md` |
| 4 | `pr-comment-poster` | PR reference + Stage 3 comments, unchanged | Pending review with line comments |

Keep the clone path from Stage 1, and the Stage 2 Verification Trace and Scope Contract in your own context. The final report needs them.

### Stage gates

- **Stage 1 returns nothing usable** (PR not found, no access, clone/checkout failed, empty diff) → ✘, cleanup, stop.
- **Stage 2 returns zero findings** → report it as a clean result, skip Stages 3–4. Do not open an empty pending review.
- **Stage 3 drops findings** → continue with what remains. If nothing remains, skip Stage 4.
- **Stage 4 skips comments** (line outside the diff) → keep them for the report, with the reason.
- **Transient failure** (timeout, rate limit, 5xx) → retry that stage once with identical input. A second failure → ✘, stop. A stage that fails on content (bad PR, denied access) is never retried.
- **Cleanup always runs** once Stage 1 created a clone: after Stage 4, or whenever the pipeline stops early (zero findings, ✘, user stops).
- Never continue with a stage's partial or malformed output, and never invent a result to keep the pipeline moving.

## Cleanup (orchestrator, after Stage 4)

Delete the clone folder reported in Stage 1's `## Clone → Path`: `rm -rf "<path>"`. Only if the path is inside this project's `temp/` and the folder name starts with `pr-` — never `temp/` itself, `temp/.gitkeep`, or anything else. Print `🧹 Folder clone dihapus — <path>`, or ✘ with the error and the path so the user can delete it manually.

## Narration (Bahasa Indonesia, one line each, no extra prose)

Print before and after every stage:

```
▶ Stage N/4 — <nama stage> (<agent>)...
✔ Stage N/4 selesai — <concrete result from the stage's real output>
```

Use ✘ on failure. Example results: `branch feat/x di-clone ke temp/pr-repo-12-20261002-101500, 5 file berubah`, `3 temuan lolos gate`, `3 komentar final disiapkan`, `3/3 komentar diposting`.

## Hard rules

- The review stays **pending**. Never submit, approve, or request changes, and never call a tool with `submit_pending` or an `event`.
- No summary comment unless the user explicitly asks.
- All PR-facing text is Bahasa Indonesia. If a comment comes back in another language, send it back to Stage 3 for rewriting before posting.
- Comments land only on lines added, modified, or deleted in the PR diff.
- The clone is read-only review material: never modify, commit, or push from it.

## Final report (Bahasa Indonesia)

- Findings that passed verification vs. comments actually posted.
- Link/ID of the pending review, for the user to submit manually.
- Comments skipped in Stage 4, with reasons.
- Verification Trace summary: callees verified vs. `unverified` (list each `unverified` explicitly), new identifiers checked for reuse, and any code path that could not be traced. This tells the user how deep the review really went.
- Scope Contract and Phase 8 result: the source of intent (issue #N, PR description, or inferred from the diff), hunks classified out of scope, and unmet acceptance criteria. If everything is Inti/Pendukung and all criteria are met, say so explicitly.
- Cleanup result (clone folder deleted, or the path left behind and why).
- Pre-flight notes, if any (draft PR, continued after a closed PR).
