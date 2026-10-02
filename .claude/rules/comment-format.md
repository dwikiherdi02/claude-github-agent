# Comment Format Rules

All PR review comments and issue content must follow these format rules. Instructional text in this file is in English; all actual GitHub-facing output (comment bodies, issue titles and bodies) must be written in Bahasa Indonesia.

---

## PR Review Comment Format

Every review comment must follow this exact structure. The comment body must be written in Bahasa Indonesia.

**Standard format (without package recommendation):**

````
[SEVERITY] Short problem title

**Masalah:**
Describe concretely what is wrong in this line/block of code.

**Mengapa Bermasalah:**
Describe the impact or risk — what can happen if left unaddressed.

**Saran Perbaikan:**
```suggestion
corrected replacement code
place here — this replaces the commented lines
```
````

---

**Format with package recommendation** *(use when the fix requires a third-party library)*:

````
[SEVERITY] Short problem title

**Masalah:**
Describe concretely what is wrong.

**Mengapa Bermasalah:**
Describe the impact or risk.

**Saran Perbaikan:**
```suggestion
// implementation using the recommended package
replacement_code_here();
```

**Rekomendasi Package:**
| Atribut | Detail |
|---|---|
| **Package** | `package-name` |
| **Registry** | npm / packagist / pip / gem / go modules / etc |
| **Fungsi** | What this package does in the context of this fix |
| **Status Keamanan** | ✅ Aman / ⚠️ Perlu Perhatian / ❌ Tidak Disarankan |
| **Keterangan Keamanan** | [Reason for status: maintainer, CVE, update frequency, download count, org backing] |
| **Versi Disarankan** | `x.y.z` or `^x.y.z` |
| **Alternatif** | `alternative-name` — [reason if primary is ⚠️ or ❌] |

> ⚠️ **Setelah instalasi**, jalankan `[ecosystem audit command]` untuk memverifikasi tidak ada kerentanan baru:
> - npm: `npm audit`
> - Composer: `composer audit`
> - pip: `pip-audit`
> - Go: `govulncheck ./...`
> - Gem: `bundle audit`
> - Maven/Gradle: `mvn dependency-check:check` / `gradle dependencyCheckAnalyze`
````

---

**"Fix Suggestion" Rules — Suggested Changeset:**

Use the ` ```suggestion ` format (GitHub Suggested Changeset) so reviewers can click **Commit suggestion** or **Add suggestion to batch** directly without manual copy-paste.

```
```suggestion
replacement code here
```
```

Usage rules:
- The suggestion block content is the **replacement code** for the commented lines — do NOT include "Sebelum/Setelah" labels inside the suggestion block
- GitHub automatically shows the diff between the original code and the suggestion
- Use suggestion blocks for fixes that are a direct replacement of the commented line(s)/block
- **If the fix cannot be expressed as a single suggestion** (cross-file, architectural changes, or changes in many places), fall back to a regular code block with "Sebelum" / "Setelah" labels and explain the full scope
- If the fix can be done natively/built-in OR with a package, show both in the suggestion or explanation and recommend which is more appropriate for the codebase context

---

## Severity Reference

### [KRITIS]
Use for:
- Security vulnerabilities that can be exploited
- Changes that break previously correct function behavior
- Data loss risks
- Race conditions that can cause data corruption

**Example (without package):**

````
[KRITIS] SQL Injection pada query pencarian user

**Masalah:**
Input dari pengguna dimasukkan langsung ke dalam query SQL tanpa prepared statement.

**Mengapa Bermasalah:**
Penyerang dapat memanipulasi query untuk mengekstrak seluruh data dari database,
menghapus tabel, atau bypass autentikasi.

**Saran Perbaikan:**
```suggestion
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = ?");
$stmt->execute([$email]);
$result = $stmt->fetchAll();
```
````

**Example (with package — password hashing case):**

````
[KRITIS] Password di-hash menggunakan MD5 yang sudah tidak aman

**Masalah:**
MD5 bukan algoritma yang cocok untuk hashing password — sangat cepat sehingga
mudah di-brute-force dan rentan terhadap rainbow table attack.

**Mengapa Bermasalah:**
Jika database bocor, password pengguna dapat di-crack dalam hitungan menit
menggunakan GPU modern atau layanan online cracking.

**Saran Perbaikan:**
Gunakan bcrypt bawaan PHP — tidak perlu package tambahan:

```suggestion
$hash = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);
```

Untuk verifikasi: `password_verify($inputPassword, $hash)`

Jika ingin menggunakan library untuk manajemen autentikasi yang lebih lengkap:

**Rekomendasi Package:**
| Atribut | Detail |
|---|---|
| **Package** | `firebase/php-jwt` |
| **Registry** | Packagist (Composer) |
| **Fungsi** | Pembuatan dan validasi JWT untuk autentikasi stateless |
| **Status Keamanan** | ✅ Aman |
| **Keterangan Keamanan** | Dikelola oleh Google Firebase, >3,000 GitHub stars, rilis aktif, tidak ada CVE aktif, digunakan secara luas di ekosistem PHP |
| **Versi Disarankan** | `^6.10` |
| **Alternatif** | `lcobucci/jwt` — lebih strict typing, cocok untuk proyek yang menggunakan PHP 8+ strict mode |

> ⚠️ **Setelah instalasi**, jalankan:
> ```
> composer audit
> ```
````

---

### [MAJOR]
Use for:
- N+1 queries that will cause production performance problems
- DRY violations that increase maintenance burden
- Missing error handling that causes silent failures
- Changes to existing working code without justification

**Example (without package — N+1, multi-line fix so uses regular code block):**

````
[MAJOR] N+1 query pada listing produk

**Masalah:**
Setiap iterasi loop memanggil `$product->category` yang memicu query database baru.
Jika ada 100 produk, ini akan memicu 101 query (1 untuk daftar + 100 untuk kategori).

**Mengapa Bermasalah:**
Pada production dengan ratusan atau ribuan produk, ini akan menyebabkan
response time yang sangat lambat dan beban tinggi pada database.

**Saran Perbaikan:**
Fix ini melibatkan perubahan pada baris query dan baris loop. Ganti baris query:

```suggestion
$products = Product::with('category')->get();
```

Setelah perubahan ini, `$product->category->name` di dalam loop tidak akan memicu
query tambahan karena relasi sudah di-eager-load.
````

**Example (with package — input validation case):**

````
[MAJOR] Validasi input dilakukan manual dengan puluhan kondisi if-else yang duplikat

**Masalah:**
Logika validasi ditulis ulang di setiap controller (UserController, OrderController,
ProductController) dengan pola yang sama namun sedikit berbeda. Ini DRY violation
sekaligus menyulitkan perubahan aturan validasi di masa depan.

**Mengapa Bermasalah:**
Ketika aturan validasi berubah, harus diupdate di banyak tempat dan rawan
terlewat, yang menyebabkan inkonsistensi validasi antar endpoint.

**Saran Perbaikan:**
Ganti blok validasi manual ini dengan schema validation terpusat:

```suggestion
const schema = Joi.object({
  email: Joi.string().email().required(),
  name: Joi.string().min(2).required(),
});
const { error } = schema.validate(req.body);
if (error) return res.status(400).json({ error: error.details[0].message });
```

**Rekomendasi Package:**
| Atribut | Detail |
|---|---|
| **Package** | `joi` |
| **Registry** | npm |
| **Fungsi** | Schema validation untuk object JavaScript/TypeScript |
| **Status Keamanan** | ✅ Aman |
| **Keterangan Keamanan** | Dikelola oleh Hapi.js team (Sideway), >20 juta download/minggu di npm, aktif diperbarui, audit bersih, tidak ada CVE aktif |
| **Versi Disarankan** | `^17.13` |
| **Alternatif** | `zod` — lebih cocok jika project menggunakan TypeScript karena inferensi tipe lebih baik |

> ⚠️ **Setelah instalasi**, jalankan:
> ```
> npm audit
> ```
````

---

### [MINOR]
Use for:
- Naming inconsistencies against existing codebase conventions
- Unhandled edge cases with limited impact
- Code structure that does not follow existing patterns

**Example:**

````
[MINOR] Penamaan tidak konsisten dengan konvensi yang ada

**Masalah:**
Fungsi ini dinamai `getUserData()` sedangkan fungsi serupa lainnya di codebase
menggunakan pola `fetchUser()`, `fetchOrder()`, `fetchProduct()` (lihat: `services/UserService.js:34`).

**Mengapa Bermasalah:**
Inkonsistensi naming membuat codebase lebih sulit dipahami dan dipelihara —
developer baru tidak tahu mana konvensi yang benar.

**Saran Perbaikan:**
```suggestion
function fetchUserData() {
```

Semua pemanggil fungsi ini juga perlu diupdate agar konsisten.
````

---

### [SARAN]
Use for:
- More idiomatic patterns for the language/framework in use
- Minor optimizations with no significant impact
- More readable alternatives that are not mandatory

**Example (without package):**

````
[SARAN] Bisa disederhanakan dengan optional chaining

**Masalah:**
Pengecekan nested ini cukup verbose.

**Mengapa Bermasalah:**
Tidak bermasalah secara fungsional, namun ada cara yang lebih ringkas
dan lebih idiomatis di JavaScript modern (ES2020+).

**Saran Perbaikan:**
```suggestion
const name = user?.profile?.name;
```
````

**Example (with package — date formatting suggestion):**

````
[SARAN] Parsing dan formatting tanggal dilakukan manual dengan string manipulation

**Masalah:**
Tanggal diformat secara manual menggunakan substring dan concatenation.
Ini berfungsi untuk kasus sederhana tapi rawan error untuk timezone,
locale berbeda, atau format tanggal yang lebih kompleks.

**Mengapa Bermasalah:**
Tidak kritis, namun pendekatan ini tidak scalable dan sulit di-maintain
jika ada kebutuhan format tanggal yang beragam di masa depan.

**Saran Perbaikan:**
Gunakan `Intl` bawaan browser/Node — tanpa package tambahan:

```suggestion
const formatted = new Intl.DateTimeFormat('id-ID', {
  year: 'numeric', month: '2-digit', day: '2-digit'
}).format(date);
```

Jika butuh manipulasi tanggal yang lebih kompleks, pertimbangkan package berikut:

**Rekomendasi Package:**
| Atribut | Detail |
|---|---|
| **Package** | `date-fns` |
| **Registry** | npm |
| **Fungsi** | Utility functions untuk parsing, formatting, dan manipulasi tanggal |
| **Status Keamanan** | ✅ Aman |
| **Keterangan Keamanan** | >16 juta download/minggu, aktif diperbarui, tree-shakeable (hanya import fungsi yang dipakai), tidak ada CVE aktif, komunitas besar |
| **Versi Disarankan** | `^3.6` |
| **Alternatif** | `dayjs` — lebih ringan (~2KB) jika hanya butuh operasi dasar; hindari `moment.js` karena sudah deprecated dan bundle size besar |

> ⚠️ **Setelah instalasi**, jalankan:
> ```
> npm audit
> ```
````

---

## Summary Comment Format *(Optional — only when explicitly requested)*

The review summary is **not posted automatically**. Only create and post a summary comment when the user explicitly requests it (e.g., "give a review summary", "show summary"). Use this template:

```markdown
## Ringkasan Review

**Total Temuan:**
- [KRITIS]: X temuan
- [MAJOR]: X temuan
- [MINOR]: X temuan
- [SARAN]: X temuan

**Dependensi Baru yang Ditambahkan:** *(hapus bagian ini jika tidak ada)*
| Package | Status | Keterangan |
|---|---|---|
| `nama-package` | ✅ Aman | Actively maintained, no CVE |
| `nama-package` | ⚠️ Perlu Perhatian | CVE minor sudah di-patch di versi terbaru |
| `nama-package` | ❌ Tidak Disarankan | Tidak diperbarui sejak 2022, CVE belum di-patch |

**Masalah Utama yang Perlu Diperhatikan:**
1. [Temuan paling penting pertama — file:baris]
2. [Temuan paling penting kedua — file:baris]
3. [Temuan paling penting ketiga — file:baris]

**Konteks Review:**
[1-2 kalimat tentang apa yang direview, tujuan PR ini, dan area kode yang terpengaruh]

**Rekomendasi:**
[Apakah PR perlu perbaikan sebelum merge? Sebutkan secara eksplisit temuan mana yang WAJIB diselesaikan (KRITIS/MAJOR) vs yang bisa dilakukan setelah merge (MINOR/SARAN)]

---
*Review ini dibuat secara otomatis. Harap periksa setiap komentar sebelum disubmit.*
```

---

## What NOT to Comment On

Do not post a comment for:
- Personal style preferences with no objective impact
- Alternative implementations that are equally valid
- Code that is different from what YOU would write but follows the project's existing patterns
- Anything that is already commented on by other reviewers in existing comments
- Trivial whitespace or formatting if the project has an auto-formatter configured
