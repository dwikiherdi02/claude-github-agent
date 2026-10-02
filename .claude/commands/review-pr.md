Review the following pull request: $ARGUMENTS

Run this review as a 4-stage pipeline. Execute the stages **in order**, each as a separate `Task`/Agent call to the named subagent, feeding each stage's full output as input to the next. Do not skip a stage or merge two stages into one call. This runs synchronously in the normal manual review cycle — do not schedule it as a background loop.

**Narration requirement:** print a short status line to the user *before* each stage starts and *after* it finishes — this is the only way the user can see the pipeline actually progressing stage-by-stage instead of one opaque block. Use this exact shape (Bahasa Indonesia, one line each, no extra prose around them):

```
▶ Stage 1/4 — Collect (pr-data-collector, Haiku)...
✔ Stage 1/4 selesai — <1-line concrete result, e.g. "5 file terkumpul, 2 komentar existing ditemukan">
```

Repeat that ▶/✔ pair for every stage (2/4, 3/4, 4/4) with the result line specific to that stage's actual output (e.g. Stage 2: jumlah temuan lolos gate; Stage 3: jumlah komentar final disiapkan; Stage 4: jumlah komentar berhasil diposting + link pending review). If a stage returns nothing useful (e.g. zero findings) or fails, say so plainly in the ✔/✘ line instead of skipping it — do not fabricate a result to keep the narration looking clean. Use ✘ instead of ✔ if a stage errors out, and stop the pipeline there rather than continuing to the next stage with bad input.

---

## Stage 1 — Collect (`pr-data-collector`, Haiku)

Call subagent `pr-data-collector` with the PR reference ($ARGUMENTS). It returns raw PR metadata, **the full linked issue** (the authority on what this PR is supposed to do), full diffs, full file contents, existing comments, the **actual definitions of every external function/constant the changed lines call**, **reuse-candidate search hits** for newly added identifiers, and whether a test suite exists. Pure data collection, no analysis.

## Stage 2 — Analyze (`pr-code-analyzer`, Sonnet 5, high effort, manual cycle)

Call subagent `pr-code-analyzer`, passing it Stage 1's full output verbatim. It applies CLAUDE.md Rules 1-7, all eight phases of the Mandatory Pre-Review Exploration Protocol (Phase 1 Scope Contract from the linked issue, Phase 3B callee validation, Phase 6 reuse search, Phase 7 correctness trace, Phase 8 scope alignment), and the Pre-Comment Verification Gate, and returns a **Verification Trace** plus a list of verified findings (or an explicit "no findings" result). This is the expensive, thorough stage — let it take the time it needs.

Pass only the *findings* to Stage 3 — the Verification Trace is internal and must not reach the PR.

## Stage 3 — Write Feedback (`pr-feedback-writer`, Sonnet 5, medium effort, manual cycle)

Call subagent `pr-feedback-writer`, passing it Stage 2's findings verbatim. It formats each finding into a final GitHub review comment per `.claude/rules/comment-format.md` (severity tag, Masalah/Mengapa Bermasalah/Saran Perbaikan, suggestion blocks, package security write-ups where relevant) — all in Bahasa Indonesia.

## Stage 4 — Post as Pending (`pr-comment-poster`, Haiku)

Call subagent `pr-comment-poster`, passing it the PR reference and Stage 3's finished comment list. It opens a pending review on the PR and adds each comment as an individual line comment. It must never submit, approve, or request changes — the review stays **pending** for a human to submit.

---

## After the pipeline finishes

Report to the user, in Bahasa Indonesia:
- Berapa banyak temuan yang lolos verifikasi dan berapa yang diposting.
- Link/ID review yang masih **pending**.
- Temuan yang di-skip pada Stage 4 (jika ada) beserta alasannya.
- Ringkasan Verification Trace dari Stage 2: berapa callee tervalidasi vs. `unverified`, berapa identifier baru yang dicek reuse-nya, dan apakah ada jalur logika yang tidak bisa ditelusuri. Sebutkan setiap `unverified` secara eksplisit — ini yang memberi tahu user seberapa dalam review-nya benar-benar berjalan.
- **Scope Contract dan hasil Phase 8**: issue mana yang jadi acuan (atau "tujuan disimpulkan dari diff" jika tidak ada issue tertaut), hunk mana saja yang dinilai di luar scope, dan kriteria penerimaan issue mana yang belum terpenuhi. Kalau semua hunk Inti/Pendukung dan semua kriteria terpenuhi, katakan itu secara eksplisit — bukan didiamkan.

## Hard constraints (apply across all stages)

- **Never** call any tool that submits/finalizes/approves/requests-changes on the review — only individual pending comments.
- **Never** post a summary comment unless the user explicitly asked for one.
- Every PR-facing comment must be in Bahasa Indonesia.
- Only comment on lines actually added/modified/deleted in this PR's diff.
