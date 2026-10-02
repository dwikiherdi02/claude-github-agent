# Riwayat Versi

## v1.0.0 — 2026-10-02

### Ditambahkan
- Rilis awal workspace Claude Code untuk agen GitHub (Code Review & Issue Agent). Isinya murni konfigurasi: instruksi, command, subagent, dan izin tool — tanpa kode sumber aplikasi.
- `CLAUDE.md`: identitas agent, Rule 1–7 (keamanan, performa, arsitektur, preservasi perilaku, error handling, saran fix & rekomendasi package, kebenaran & kontrak lintas file), protokol eksplorasi 8 fase, dan larangan keras.
- Command: `/review-pr`, `/create-issue`, `/fix-issue`, `/create-pr`, dan `/check-secrets`.
- Subagent pipeline review PR (`pr-data-collector`, `pr-code-analyzer`, `pr-feedback-writer`, `pr-comment-poster`) beserta skill `pr-review-pipeline` yang menjalankannya berurutan dan menyimpan hasilnya sebagai review **pending**.
- Rules pendukung: `comment-format.md`, `review-checklist.md`, `issue-to-pr-workflow.md`, dan `scope-limits.md`.
- `.claude/settings.json`: daftar izin tool GitHub MCP (allow/deny); aksi merge, push file via API, fork, dan sejenisnya diblokir.
- Dokumentasi: `README.md`, `GUIDE.md` berisi contoh prompt, serta lisensi (`LICENSE`).
- Folder `attachment/`, `guide-issues/`, dan `temp/` (isinya di-ignore, hanya `.gitkeep` yang dilacak).
