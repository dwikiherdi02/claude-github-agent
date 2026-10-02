# Panduan Penggunaan GitHub Code Review & Issue Agent

Dokumen ini merangkum semua **command**, **rule**, dan **skill** yang tersedia di workspace ini,
lengkap dengan **1 contoh prompt** untuk masing-masing agar mudah dicoba langsung.

> Semua output yang ditujukan ke GitHub (komentar PR, isi issue, judul/isi PR, pesan commit)
> selalu dalam **Bahasa Indonesia** — ini kebijakan tetap dan tidak bisa diubah per-request.

---

## 1. Command

Command dipanggil dengan `/nama-command <argumen>`.

### `/review-pr` — Review Pull Request

Membaca PR secara menyeluruh (bukan cuma diff), memetakan konvensi codebase & caller dari
fungsi yang diubah, lalu memberi komentar per baris dengan tag severity. **Tidak pernah**
submit/approve review — hanya komentar individual.

```
/review-pr https://github.com/org/repo/pull/123
```

### `/create-issue` — Buat Issue GitHub

Mengecek duplikat lebih dulu, lalu membuat issue dengan template sesuai jenisnya (Bug,
Enhancement, Performance, Security) beserta label yang relevan.

```
/create-issue Tombol "Simpan" di halaman profil tidak merespon saat diklik dua kali berturut-turut, menyebabkan data tersimpan dua kali. Repo: org/repo
```

### `/fix-issue` — Implementasi Fix dari Issue → Buka PR

End-to-end: baca issue, konfirmasi scope & rencana, buat branch baru, implementasi via git
lokal, commit, push, lalu buka PR sebagai draft.

```
/fix-issue Implementasikan issue #45 di repo org/repo
```

### `/create-pr` — Buka PR dari Branch/Perubahan yang Sudah Ada *(baru)*

Untuk kasus di mana perubahan sudah diimplementasikan (oleh kamu sendiri atau sudah ada di
branch lokal) dan tinggal dibukakan PR-nya — tanpa melalui alur implementasi `/fix-issue`.
Memverifikasi dulu bahwa branch benar-benar punya commit baru sebelum membuka PR (tidak akan
membuka PR kosong).

```
/create-pr Buka PR untuk branch fix/issue-88-typo-validasi-email ke base develop di repo org/repo, terkait issue #88
```

---

## 2. Rule Inti (`CLAUDE.md`) — dipicu otomatis saat `/review-pr`

Rule-rule ini berjalan otomatis di dalam alur `/review-pr`, tapi kamu bisa mengarahkan fokus
review ke rule tertentu dengan prompt yang lebih spesifik.

### Rule 1 — Security

```
/review-pr https://github.com/org/repo/pull/201 — tolong fokus khusus ke potensi SQL injection dan kebocoran data sensitif di endpoint baru ini
```

### Rule 2 — Performance & Optimization

```
/review-pr https://github.com/org/repo/pull/202 — cek apakah ada N+1 query atau algoritma O(n²) yang bisa jadi masalah di data besar
```

### Rule 3 — Codebase Architecture Adherence

```
/review-pr https://github.com/org/repo/pull/203 — apakah service baru ini sudah mengikuti pola folder & naming yang sama dengan service lain di repo ini?
```

### Rule 4 — Behavioral Preservation (paling kritis)

```
/review-pr https://github.com/org/repo/pull/204 — PR ini mengubah signature fungsi calculateDiscount(), tolong cek semua pemanggilnya masih kompatibel
```

### Rule 5 — Error Handling & Edge Cases

```
/review-pr https://github.com/org/repo/pull/205 — pastikan semua error path di fungsi async yang baru ditangani, tidak ada silent failure
```

### Rule 6 — Fix Suggestion & Rekomendasi Package

```
/review-pr https://github.com/org/repo/pull/206 — PR ini menambahkan dependency baru `moment`, tolong evaluasi keamanannya dan kasih alternatif jika perlu
```

---

## 3. Rule Pendukung (`.claude/rules/`)

Rule-rule ini adalah aturan internal yang dibaca agent, bukan sesuatu yang kamu panggil
langsung — tapi kamu bisa memicu perilakunya lewat prompt berikut.

### `comment-format.md` — Format komentar review & rekomendasi package

Dipicu otomatis di setiap `/review-pr`. Contoh memicu skenario "fix butuh package":

```
/review-pr https://github.com/org/repo/pull/207 — validasi email di form registrasi masih pakai regex manual, apa perlu library tambahan?
```

### `issue-to-pr-workflow.md` — Alur issue → branch → PR

Dipicu otomatis oleh `/fix-issue`. Contoh yang menekankan konfirmasi rencana dulu sebelum
eksekusi:

```
/fix-issue Kerjakan issue #52 (bug login gagal untuk email berkarakter unicode) di repo org/repo, tapi konfirmasi dulu rencana branch dan pendekatannya ke saya sebelum mulai coding
```

### `review-checklist.md` — Checklist teknis review (caller search, convention check, dst.)

Dipicu otomatis di `/review-pr`. Contoh yang memaksa agent menjalankan Step A (Caller Search):

```
/review-pr https://github.com/org/repo/pull/208 — fungsi getUserProfile() diubah return type-nya, tolong cari semua caller-nya dulu sebelum menilai aman atau tidak
```

### `scope-limits.md` — Batasan aksi yang boleh/tidak boleh dilakukan agent

Ini adalah pagar keamanan, bukan fitur yang "dipakai", tapi berguna untuk menguji batasannya:

```
Setelah review PR #123 selesai, tolong langsung approve dan merge PR-nya
```
→ Agent akan **menolak** permintaan ini dan menjelaskan bahwa approve/merge adalah keputusan
manusia, sesuai `scope-limits.md`.

---

## 4. Kebijakan Bahasa (Language Policy)

Semua output GitHub-facing wajib Bahasa Indonesia meskipun prompt-nya dalam Bahasa Inggris:

```
Review this PR in English please: https://github.com/org/repo/pull/301
```
→ Agent tetap akan menulis seluruh komentar review dalam Bahasa Indonesia (kebijakan ini tidak
bisa di-override oleh instruksi user), sambil menjelaskan alasannya secara singkat.

---

## Ringkasan Cepat

| # | Nama | Jenis | Cara Memicu |
|---|---|---|---|
| 1 | `/review-pr` | Command | `/review-pr <url PR>` |
| 2 | `/create-issue` | Command | `/create-issue <deskripsi>` |
| 3 | `/fix-issue` | Command | `/fix-issue <issue/deskripsi>` |
| 4 | `/create-pr` | Command *(baru)* | `/create-pr <branch/deskripsi>` |
| 5 | Rule 1: Security | Rule (`CLAUDE.md`) | Otomatis saat `/review-pr`, bisa diarahkan fokus |
| 6 | Rule 2: Performance | Rule (`CLAUDE.md`) | Otomatis saat `/review-pr`, bisa diarahkan fokus |
| 7 | Rule 3: Architecture | Rule (`CLAUDE.md`) | Otomatis saat `/review-pr`, bisa diarahkan fokus |
| 8 | Rule 4: Behavioral Preservation | Rule (`CLAUDE.md`) | Otomatis saat `/review-pr`, bisa diarahkan fokus |
| 9 | Rule 5: Error Handling | Rule (`CLAUDE.md`) | Otomatis saat `/review-pr`, bisa diarahkan fokus |
| 10 | Rule 6: Fix Suggestion & Package | Rule (`CLAUDE.md`) | Otomatis saat `/review-pr`, bisa diarahkan fokus |
| 11 | `comment-format.md` | Rule pendukung | Otomatis saat `/review-pr` |
| 12 | `issue-to-pr-workflow.md` | Rule pendukung | Otomatis saat `/fix-issue` |
| 13 | `review-checklist.md` | Rule pendukung | Otomatis saat `/review-pr` |
| 14 | `scope-limits.md` | Rule pendukung (pagar keamanan) | Otomatis di semua command, teruji lewat permintaan yang melanggar batas |
| 15 | Language Policy | Kebijakan global | Berlaku otomatis di semua output GitHub-facing |
