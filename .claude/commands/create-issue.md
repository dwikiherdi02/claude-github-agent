Create a GitHub issue based on the following: $ARGUMENTS

Execute every step below in order.

---

## Step 1: Parse the Input

Determine what type of issue needs to be created:
- **Bug** — something is broken or behaving incorrectly
- **Enhancement / Feature** — new capability or improvement
- **Performance** — the app works but is slow or inefficient
- **Security** — a vulnerability or security concern
- **Question / Discussion** — clarification needed on expected behavior

Identify the target repository from the input. If not specified, ask before proceeding.

---

## Step 2: Research Before Creating

1. Search existing open and closed issues in the repository to avoid creating a duplicate
2. If code is mentioned, fetch the relevant files to understand the context
3. If the input references a PR or existing issue, read it fully

If a duplicate is found, report the existing issue URL instead of creating a new one.

---

## Step 3: Compose the Issue in Bahasa Indonesia

Write the issue title and body entirely in Bahasa Indonesia.

Choose the correct template based on issue type:

---

### Template: Bug Report

**Judul:** `[BUG] <deskripsi singkat masalah>`

**Body:**
```
## Deskripsi Bug

[Jelaskan bug secara singkat dan jelas. Apa yang tidak berfungsi?]

## Langkah Reproduksi

1. [Langkah pertama]
2. [Langkah kedua]
3. [Dan seterusnya...]

## Perilaku yang Diharapkan

[Apa yang seharusnya terjadi]

## Perilaku Aktual

[Apa yang sebenarnya terjadi]

## Dampak

[Seberapa besar dampaknya — apakah memblokir pengguna, menyebabkan data salah, atau hanya gangguan minor?]

## Konteks Tambahan

- Lingkungan: [production / staging / development]
- Versi: [jika relevan]
- [Informasi tambahan lain yang relevan]
```

---

### Template: Feature Request / Enhancement

**Judul:** `[ENHANCEMENT] <deskripsi singkat fitur>`

**Body:**
```
## Deskripsi Fitur

[Jelaskan fitur atau peningkatan yang diinginkan secara singkat]

## Latar Belakang / Motivasi

[Mengapa fitur ini diperlukan? Masalah apa yang diselesaikan? Siapa yang diuntungkan?]

## Acceptance Criteria

- [ ] [Kriteria pertama yang harus terpenuhi]
- [ ] [Kriteria kedua]
- [ ] [Dan seterusnya...]

## Pertimbangan Implementasi

[Hal-hal teknis yang perlu diperhatikan saat implementasi, jika ada]

## Referensi

[PR, issue, atau dokumentasi terkait, jika ada]
```

---

### Template: Performance Issue

**Judul:** `[PERFORMANCE] <deskripsi singkat masalah performa>`

**Body:**
```
## Deskripsi Masalah Performa

[Jelaskan masalah performa yang ditemukan — apa yang lambat, boros memori, atau tidak efisien]

## Lokasi Kode

- File: `[path/to/file.ext]`
- Baris: [nomor baris jika relevan]
- Fungsi/Method: `[nama fungsi]`

## Analisis Dampak

[Bagaimana ini mempengaruhi performa aplikasi — response time, memory usage, database load, dll]

## Akar Masalah

[Penjelasan teknis kenapa ini menyebabkan masalah performa — N+1 query, nested loop, dll]

## Saran Perbaikan

[Solusi yang disarankan secara konkret]
```

---

### Template: Security Issue

**Judul:** `[SECURITY] <deskripsi singkat kerentanan>`

**Body:**
```
## Jenis Kerentanan

[Jenis kerentanan — SQL Injection / XSS / CSRF / IDOR / dll]

## Tingkat Keparahan

[Critical / High / Medium / Low]

## Deskripsi

[Jelaskan kerentanan — cukup untuk dipahami tanpa memberikan exploit detail lengkap di issue publik]

## Lokasi

- File: `[path/to/file.ext]`
- Fungsi/Endpoint: `[nama atau path]`

## Dampak Potensial

[Apa yang bisa terjadi jika kerentanan ini dieksploitasi — data breach, unauthorized access, dll]

## Saran Mitigasi

[Langkah-langkah konkret yang disarankan untuk memperbaiki kerentanan ini]
```

---

## Step 4: Select Labels

Apply appropriate labels from those available in the repository:

| Issue Type | Label yang Disarankan |
|---|---|
| Bug | `bug` |
| Feature / Enhancement | `enhancement` |
| Performance | `performance` |
| Security | `security` |
| High priority | `priority: high` |
| Needs discussion | `question` |

Use only labels that exist in the repository. Check available labels before applying.

---

## Step 5: Create the Issue

Create the issue using the composed title and body.

After creating, report back:
- The issue URL
- The issue number
- A one-line summary in Bahasa Indonesia of what was created

---

## Constraints

- All issue content (title and body) must be written in **Bahasa Indonesia**
- Do not create duplicates — always search first
- Do not include sensitive information (actual credentials, exploit payloads) in issue bodies
- Do not add assignees or milestones unless explicitly requested
