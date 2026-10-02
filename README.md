# GitHub Code Review & Issue Agent

Workspace [Claude Code](https://claude.com/claude-code) yang mengubah Claude menjadi agen GitHub khusus untuk:

- **Mereview Pull Request** secara menyeluruh (keamanan, performa, arsitektur, kebenaran logika, kesesuaian scope) dan memposting temuan sebagai *komentar baris* yang masih **pending**, sehingga manusia yang memutuskan submit-nya.
- **Membuat issue** (bug, enhancement, performance, security) dengan template dan label yang sesuai, setelah mengecek duplikat.
- **Mengimplementasikan fix dari issue** di branch khusus, lalu membuka **draft PR**.
- **Membuka PR** dari branch lokal yang sudah berisi commit.

> **Kebijakan bahasa:** semua keluaran yang tampil di GitHub (komentar review, isi issue, judul/isi PR, pesan commit) **selalu Bahasa Indonesia**, apa pun bahasa prompt-nya. Ini kebijakan tetap yang tidak bisa di-override per-request.

Workspace ini **bukan aplikasi** dengan kode sumber, build, atau server. Isinya murni konfigurasi Claude Code (instruksi, command, subagent, dan izin tool). Tidak ada `npm install` atau `pip install` untuk proyek ini.

---

## Daftar Isi

1. [Struktur Workspace](#struktur-workspace)
2. [Prasyarat & Instalasi](#prasyarat--instalasi)
3. [Cara Menjalankan](#cara-menjalankan)
4. [Command](#command)
5. [Agent (Subagent)](#agent-subagent)
6. [Rules](#rules)
7. [Skill](#skill)
8. [Plugin](#plugin)
9. [MCP Server & Izin Tool](#mcp-server--izin-tool)
10. [Batasan & Keamanan](#batasan--keamanan)
11. [Troubleshooting](#troubleshooting)

---

## Struktur Workspace

```
github-agent/
├── CLAUDE.md                     # Identitas agent, Rule 1-7, protokol eksplorasi 8 fase, larangan keras
├── GUIDE.md                      # Contoh prompt untuk setiap command & rule
├── README.md                     # Dokumen ini
├── temp/                         # Clone sementara branch PR untuk /review-pr (dihapus otomatis)
└── .claude/
    ├── settings.json             # Izin tool GitHub MCP (allow/deny), dibagikan lewat repo
    ├── settings.local.json       # Pengaturan lokal: model, effort, izin tambahan
    ├── commands/                 # Slash command
    │   ├── review-pr.md
    │   ├── create-issue.md
    │   ├── fix-issue.md
    │   ├── create-pr.md
    │   └── check-secrets.md
    ├── agents/                   # Subagent pipeline review PR
    │   ├── pr-data-collector.md
    │   ├── pr-code-analyzer.md
    │   ├── pr-feedback-writer.md
    │   └── pr-comment-poster.md
    ├── skills/
    │   └── pr-review-pipeline/
    │       └── SKILL.md          # Menjalankan 4 agent di atas secara berurutan
    └── rules/                    # Aturan pendukung yang dibaca agent
        ├── comment-format.md
        ├── review-checklist.md
        ├── issue-to-pr-workflow.md
        └── scope-limits.md
```

---

## Prasyarat & Instalasi

### Wajib

| Kebutuhan | Fungsi | Cek / Instal |
|---|---|---|
| **Claude Code** (CLI atau ekstensi IDE) | Menjalankan workspace ini | Ikuti [panduan instalasi resmi](https://docs.claude.com/en/docs/claude-code/setup); cek dengan `claude --version` |
| **Git** (di Windows: Git for Windows) | Dipakai `/fix-issue` dan `/create-pr` untuk commit & push lokal, serta `/review-pr` untuk clone branch PR | `git --version` |
| **Akun GitHub + Personal Access Token (PAT)** | Autentikasi GitHub MCP server | Buat di *GitHub → Settings → Developer settings → Personal access tokens*. Scope minimal: `repo` (repo privat) atau akses *Pull requests*, *Issues*, *Contents* (read) bila memakai fine-grained token. `/fix-issue` dan `/create-pr` juga butuh izin push ke repo target |
| **GitHub CLI (`gh`)** + `gh auth login` | `/review-pr` meng-clone branch PR (`gh repo clone`, `gh pr checkout`); juga dipakai skill `pr-feedback-resolver` | `gh --version`, lalu `gh auth login` |
| **GitHub MCP server** | Semua tool `mcp__github__*` (baca PR, komentar, buat issue/branch/PR) | Lihat langkah di bawah |

### Daftarkan GitHub MCP server

Workspace ini memakai server MCP GitHub resmi (remote, HTTP). Daftarkan sekali di level user:

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp \
  --header "Authorization: Bearer <GITHUB_PAT_ANDA>"
```

Verifikasi:

```bash
claude mcp list        # server "github" harus berstatus connected
```

> **Jangan** menulis token ke file yang di-commit (`.mcp.json`, `settings.json`, README, dsb.). Simpan token hanya di konfigurasi user atau environment variable, dan cabut/ganti token bila pernah terlihat di log atau chat.

### Opsional

| Kebutuhan | Dipakai untuk |
|---|---|
| **Python 3** | `settings.local.json` mengizinkan `Bash(python3 *)` bila diperlukan skrip bantu |
| **Plugin Claude Code** | Lihat [Plugin](#plugin). Tidak ada yang wajib untuk command bawaan workspace ini |

### Clone repo target (khusus `/fix-issue` & `/create-pr`)

MCP GitHub yang terhubung **tidak punya tool tulis-file jarak jauh**. Perubahan kode dibuat lewat git lokal, jadi repo yang akan dikerjakan harus sudah di-clone di mesin Anda:

```bash
git clone https://github.com/<org>/<repo>.git
```

`/create-issue` tidak butuh clone. `/review-pr` meng-clone branch PR sendiri ke `temp/` dan menghapusnya setelah selesai, jadi Anda tidak perlu clone manual.

---

## Cara Menjalankan

```bash
cd <path-ke-folder-workspace-ini>
claude
```

Lalu ketik salah satu command di dalam sesi Claude Code:

```
/review-pr https://github.com/org/repo/pull/123
```

Contoh prompt lengkap per command dan per rule ada di [`GUIDE.md`](GUIDE.md).

---

## Command

Command berada di `.claude/commands/` dan dipanggil dengan `/nama-command <argumen>`.

| Command | Fungsi | Contoh |
|---|---|---|
| `/review-pr` | Menjalankan **pipeline 4 tahap** (clone → analyze → write → post) dan menghasilkan komentar baris **pending** pada PR, lalu menghapus folder clone di `temp/`. Tidak pernah submit, approve, atau request changes | `/review-pr https://github.com/org/repo/pull/123` |
| `/create-issue` | Mencari duplikat, memilih template (Bug / Enhancement / Performance / Security), memilih label yang ada di repo, lalu membuat issue dalam Bahasa Indonesia | `/create-issue Tombol "Simpan" tidak merespon saat diklik dua kali. Repo: org/repo` |
| `/fix-issue` | Alur end-to-end: baca issue → eksplorasi konvensi → **konfirmasi rencana** → buat branch → implementasi via git lokal → commit & push → buka **draft PR** | `/fix-issue Implementasikan issue #45 di repo org/repo` |
| `/create-pr` | Membuka PR dari branch yang sudah punya commit. Memverifikasi ada commit di depan base (tidak pernah membuka PR kosong) dan meminta konfirmasi sebelum membuka | `/create-pr Buka PR untuk branch fix/issue-88-typo ke base develop di repo org/repo, terkait issue #88` |
| `/check-secrets` | Memeriksa token, API key, password, dan private key **sungguhan** di file yang akan di-commit (sesuai `.gitignore`) dan di riwayat git. Hanya membaca, tidak mengubah apa pun, dan tidak pernah menampilkan nilai rahasianya. Jalankan sebelum commit/push | `/check-secrets` |

---

## Agent (Subagent)

Empat subagent di `.claude/agents/` membentuk pipeline `/review-pr`. Setiap tahap dipanggil berurutan, dengan status `▶ Stage n/4` dan `✔ Stage n/4 selesai` ditampilkan ke pengguna.

| Tahap | Agent | Model | Tool yang diizinkan | Tugas |
|---|---|---|---|---|
| 1 | `pr-data-collector` | Haiku | `Bash` (`gh`, `git`), `pull_request_read`, `issue_read`, `list_commits` | Meng-clone **branch PR** (bukan branch tujuan) ke `temp/pr-<repo>-<nomor>-<timestamp>/`, lalu mengembalikan path clone, metadata PR, **issue tertaut**, diff terhadap base, komentar existing, dan commit. Tanpa analisis |
| 2 | `pr-code-analyzer` | Sonnet (effort tinggi) | `Read`, `Grep`, `Glob`, `Bash` (git read-only), `pull_request_read`, `issue_read` | Analisis mendalam atas kode di folder clone: kode yang berubah (berdasarkan diff dan deskripsi PR/issue) plus setiap fungsi/service/helper di file lain yang dipanggilnya, dengan Rule 1-7, 8 fase protokol eksplorasi, dan *Pre-Comment Verification Gate*. Menghasilkan **Verification Trace** dan temuan terverifikasi |
| 3 | `pr-feedback-writer` | Sonnet (effort sedang) | `Read`, `Grep`, `Glob` | Memformat temuan menjadi komentar final sesuai `comment-format.md` (tag severity, Masalah / Mengapa Bermasalah / Saran Perbaikan, blok ` ```suggestion `, evaluasi keamanan package) |
| 4 | `pr-comment-poster` | Haiku | `pull_request_read`, `pull_request_review_write`, `add_comment_to_pending_review` | Membuat *pending review* dan menambahkan tiap komentar sebagai komentar baris. **Tidak** men-submit |

Setelah Stage 4 (atau bila pipeline berhenti lebih awal setelah clone dibuat), orchestrator menghapus folder clone di `temp/` (**cleanup**). Penghapusan hanya untuk folder `temp/pr-*`.

Laporan akhir pipeline mencakup: jumlah temuan lolos verifikasi vs. terposting, ID review pending, ringkasan Verification Trace (callee tervalidasi vs. `unverified`), Scope Contract dan hasil Phase 8, serta hasil cleanup.

---

## Rules

### Rule inti di `CLAUDE.md`

| Rule | Fokus |
|---|---|
| **Rule 1: Security** | Injection, XSS/CSRF, bypass auth, IDOR, kebocoran data sensitif, validasi input, deserialisasi, dependensi rentan |
| **Rule 2: Performance** | N+1 query, kondisi/nesting berlebih, pelanggaran DRY, kompleksitas algoritma, manajemen resource |
| **Rule 3: Architecture Adherence** | Struktur folder, penamaan, pola desain, abstraksi yang sudah ada, penempatan layer |
| **Rule 4: Behavioral Preservation** *(prioritas tertinggi)* | Signature, return value, dan side effect fungsi yang sudah bekerja tidak boleh berubah tanpa alasan |
| **Rule 5: Error Handling & Edge Cases** | Tidak ada silent failure, cek null, nilai batas, error async |
| **Rule 6: Fix Suggestion & Rekomendasi Package** | Setiap temuan wajib punya saran konkret. Package pihak ketiga dievaluasi dengan status ✅ / ⚠️ / ❌ dan wajib ada pengingat audit |
| **Rule 7: Correctness & Cross-File Contract** | Kebenaran logika, validitas kontrak antar-file, pencarian reuse, cakupan test (bila repo punya test suite) |

`CLAUDE.md` juga memuat **Protokol Eksplorasi 8 Fase** (Intent & Scope, baca file penuh, peta dependensi caller/callee, verifikasi konvensi, konteks performa, pencarian reuse, trace kebenaran, kesesuaian scope), **Pre-Comment Verification Gate** (confidence minimal 90%), dan **Hard Prohibitions**.

Tag severity komentar: `[KRITIS]`, `[MAJOR]`, `[MINOR]`, `[SARAN]`.

### Rule pendukung di `.claude/rules/`

| File | Isi |
|---|---|
| `comment-format.md` | Format baku komentar review (dengan/tanpa rekomendasi package), contoh per severity, template ringkasan opsional |
| `review-checklist.md` | Prosedur Step A-I (caller search, verifikasi konvensi, konteks performa, validasi suggestion, validasi callee, pencarian reuse, correctness trace, test coverage, scope alignment), pola keamanan, dan checklist keamanan dependensi |
| `issue-to-pr-workflow.md` | Alur issue → branch → PR: konvensi nama branch `<type>/issue-<nomor>-<slug>`, template body PR, aturan keselamatan |
| `scope-limits.md` | Tabel aksi yang boleh dan dilarang, serta kebijakan submission review dan kebijakan bahasa |

---

## Skill

### Skill lokal

| Skill | Fungsi |
|---|---|
| `pr-review-pipeline` | Menjalankan pipeline 4 agent (`pr-data-collector` → `pr-code-analyzer` → `pr-feedback-writer` → `pr-comment-poster`) secara berurutan pada sebuah PR (branch PR di-clone ke `temp/`, lalu dihapus di akhir) dan menyimpan hasilnya sebagai **pending review**. Aktif otomatis saat Anda meminta review PR dengan agent atau memberi link/nomor PR. Aturan kerasnya sama dengan `/review-pr`: tidak submit, approve, atau request changes, dan tidak memposting summary kecuali diminta |

### Skill global (opsional)

Ada **skill global opsional** yang terpasang di level user (`~/.claude/skills/` dan `~/.claude/agents/pr-feedback-resolver/`), bukan bagian dari repo ini:

| Skill | Fungsi |
|---|---|
| `pr-feedback-resolver` | Kebalikan dari `/review-pr`: mengambil komentar reviewer yang **belum resolved**, menilai relevansinya, mengimplementasikan perbaikan, menyusun balasan, lalu memposting setelah konfirmasi |

Pipeline-nya memakai empat subagent global: `pr-feedback-fetcher` (Haiku, `gh` CLI), `pr-feedback-analyzer` (Sonnet, satu-satunya yang mengedit kode), `pr-feedback-drafter` (Sonnet, hanya teks), dan `pr-feedback-poster` (Haiku, `gh` CLI). Butuh `gh` yang sudah login (`gh auth login`).

Untuk memakainya di mesin lain, salin folder `skills/pr-feedback-resolver` dan `agents/pr-feedback-resolver` ke `~/.claude/`.

---

## Plugin

**Tidak ada plugin yang wajib** untuk command dan pipeline bawaan workspace ini; semuanya berjalan dengan Claude Code, GitHub MCP, dan git.

Plugin berikut aktif di konfigurasi user pembuat workspace ini dan **opsional** sebagai pelengkap (dari marketplace `claude-plugins-official`, kecuali disebutkan lain):

| Plugin | Manfaat untuk workflow ini |
|---|---|
| `code-review` | Review diff/PR dengan tingkat effort yang bisa dipilih |
| `pr-review-toolkit` | Review PR memakai agent khusus (test coverage, silent failure, desain tipe, analisis komentar) |
| `code-simplifier` | Menyederhanakan kode setelah implementasi `/fix-issue` |
| `feature-dev` | Pengembangan fitur dengan eksplorasi codebase dan blueprint arsitektur |
| `claude-md-management` | Audit dan pembaruan `CLAUDE.md` |
| `claude-code-setup` | Rekomendasi otomasi Claude Code (hook, subagent, skill) |
| `security-guidance` | Panduan keamanan saat menulis kode |
| `frontend-design` | Panduan desain UI (tidak relevan untuk workflow review/issue) |
| `csharp-lsp`, `typescript-lsp`, `gopls-lsp`, `php-lsp` | Language server agar Claude memahami kode C#, TypeScript, Go, dan PHP saat `/fix-issue` |
| `ponytail` (marketplace `DietrichGebert/ponytail`) | Mode "solusi paling minimal"; berguna untuk menahan scope saat `/fix-issue` |

Instal contoh:

```bash
claude plugin install pr-review-toolkit@claude-plugins-official
claude plugin install code-review@claude-plugins-official
```

> Plugin LSP membutuhkan language server masing-masing terpasang di mesin (mis. `typescript-language-server`, `gopls`). Pasang hanya untuk bahasa yang repo target Anda gunakan.

---

## MCP Server & Izin Tool

### MCP server

| Server | Transport | Dipakai untuk |
|---|---|---|
| `github` | HTTP, `https://api.githubcopilot.com/mcp` | Semua operasi GitHub di workspace ini |

### `.claude/settings.json` (dibagikan lewat repo)

**Allow:** `get_me`, `pull_request_read`, `list_pull_requests`, `pull_request_review_write`, `add_comment_to_pending_review`, `add_reply_to_pull_request_comment`, `get_file_contents`, `get_commit`, `list_commits`, `list_branches`, `issue_read`, `issue_write`, `list_issues`, `search_issues`, `search_code`, `search_repositories`, `search_pull_requests`, `list_issue_types`, `list_issue_fields`, `get_label`, `create_branch`, `create_pull_request`, `update_pull_request`, `add_issue_comment`.

**Deny:** `merge_pull_request`, `delete_file`, `push_files`, `create_or_update_file`, `fork_repository`, `create_repository`, `assign_copilot_to_issue`, `create_pull_request_with_copilot`, `request_copilot_review`, `run_secret_scanning`, `update_pull_request_branch`, `delete_branch`.

### `.claude/settings.local.json` (lokal, sebaiknya tidak di-commit)

Menetapkan `model: sonnet`, `effortLevel: low`, dan mengizinkan `add_issue_comment` serta `Bash(python3 *)`. Sesuaikan dengan kebutuhan Anda.

---

## Batasan & Keamanan

Agent ini sengaja dibatasi (rinci di `.claude/rules/scope-limits.md`):

- **Tidak pernah** submit, approve, atau request-changes pada review. Hanya komentar baris pending.
- **Tidak pernah** merge PR, force-push, atau menghapus branch.
- **Tidak pernah** push ke `main` / `master` / `develop` / branch release. Selalu ke branch fitur khusus.
- **Tidak membuat** summary comment kecuali diminta eksplisit.
- **Tidak membuka** PR tanpa commit nyata di belakangnya.
- **Tidak memperluas** scope di luar yang dijelaskan issue tanpa konfirmasi.
- Komentar hanya diletakkan pada baris yang benar-benar berubah di diff, meski pembacaan kode boleh jauh lebih luas.

Catatan: pembatasan submit/approve pada review dijaga oleh **instruksi** (CLAUDE.md dan rules), bukan oleh blokir tool, karena `pull_request_review_write` ada di daftar allow. Bila ingin pengamanan teknis tambahan, pantau setiap permintaan izin yang muncul untuk tool tersebut.

---

## Troubleshooting

| Gejala | Penyebab / Solusi |
|---|---|
| Tool `mcp__github__*` tidak ditemukan atau gagal | MCP belum terdaftar atau token salah/kedaluwarsa. Jalankan `claude mcp list`, daftarkan ulang dengan PAT yang valid |
| `403` / `404` saat membaca PR atau repo privat | PAT tidak punya akses ke repo tersebut. Perbarui scope atau tambahkan repo ke fine-grained token |
| `/fix-issue` berhenti dan meminta clone | Normal: MCP tidak bisa menulis file jarak jauh. Clone repo target dulu, lalu ulangi |
| `/create-pr` menolak membuka PR | Branch tidak punya commit di depan base. Commit dan push dulu |
| Komentar review muncul dalam Bahasa Inggris | Tidak seharusnya terjadi. Kebijakan Bahasa Indonesia berlaku mutlak. Minta agent menulis ulang |
| `/review-pr` gagal di Stage 1 (clone) | `gh` belum login atau tidak punya akses ke repo. Jalankan `gh auth status` / `gh auth login` |
| Folder `temp/pr-*` tertinggal | Cleanup gagal (mis. file terkunci di Windows). Hapus manual folder yang disebut di laporan akhir |
| Komentar tidak muncul di tab *Files changed* | Review masih **pending**. Buka PR → *Review changes* untuk melihat dan men-submit-nya secara manual |

---

*Dokumen pendamping: [`GUIDE.md`](GUIDE.md) berisi contoh prompt untuk setiap command dan rule.*
