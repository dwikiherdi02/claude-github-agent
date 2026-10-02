---
description: Scan the workspace for real tokens, API keys, passwords, and private keys before commit/push
allowed-tools: Read, Grep, Glob, Bash(git ls-files:*), Bash(git log:*), Bash(git status:*), Bash(git check-ignore:*), Bash(git rev-parse:*)
---

Scan this workspace for real credentials before the user commits or pushes: $ARGUMENTS

Optional argument: a sub-path to limit the scan. Default: the whole workspace.

This command is **read-only**. Never edit, delete, stage, commit, or push anything, and never "fix" a finding yourself — report it and let the user decide. It is a local check, unrelated to the GitHub `run_secret_scanning` tool.

## 1. Build the file list

- Inside a git repo (`git rev-parse --is-inside-work-tree`): `git ls-files --cached --others --exclude-standard` — exactly what would be committed, `.gitignore` already applied.
- Not a git repo yet: Glob every file, then drop whatever `.gitignore` matches (read it; treat `dir/*` as "ignore contents" and honour `!` exceptions).
- Skip binary files.

## 2. Hard checks

- `.claude/settings.local.json` must be ignored **and** not tracked (`git check-ignore`, `git ls-files`). If it is tracked, that is a finding.
- Flag these file names if they are in the list: `.env`, `.env.*` (except `.env.example`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `id_ed25519*`, `*.keystore`.

## 3. Content patterns (Grep on the file list)

In the table, `\|` is only Markdown's escape for a regex alternation `|` — use a plain `|` in the actual pattern.

| Kind | Pattern (ripgrep) |
|---|---|
| GitHub token | `gh[pousr]_[A-Za-z0-9]{36,}` · `github_pat_[A-Za-z0-9_]{20,}` |
| Anthropic/OpenAI-style key | `sk-[A-Za-z0-9_-]{20,}` |
| AWS access key | `AKIA[0-9A-Z]{16}` |
| Google API key | `AIza[0-9A-Za-z_-]{35}` |
| Slack token | `xox[baprs]-[A-Za-z0-9-]{10,}` |
| Private key block | `-----BEGIN [A-Z ]*PRIVATE KEY-----` |
| Bearer/Basic with real value | `(?i)(bearer\|basic)\s+[A-Za-z0-9._~+/=-]{20,}` |
| URL with credentials | `://[^/\s:@]+:[^/\s@]+@` |
| Quoted assignment | `(?i)(password\|passwd\|pwd\|secret\|token\|api[_-]?key\|client[_-]?secret)["']?\s*[:=]\s*["'][^"'\s]{6,}["']` |
| Unquoted assignment (.env/ini) | `(?im)^\s*(export\s+)?[A-Z0-9_]*(PASSWORD\|SECRET\|TOKEN\|API_?KEY)[A-Z0-9_]*\s*=\s*[^\s"'$<{][^\s]{5,}` |

## 4. Triage — real credential or not?

The goal is real secrets, **not the words** "password"/"secret". Read the surrounding lines of every hit and drop:

- Prose, rules, docs, and examples that merely mention the word (e.g. the password-hashing example in `.claude/rules/comment-format.md`, `run_secret_scanning` in `settings.json`/`scope-limits.md`).
- Obvious placeholders: `<GITHUB_PAT_ANDA>`, `<TOKEN>`, `your-token-here`, `xxx`, `changeme`, `example`, `${VAR}`, `$VAR`, `process.env.X`, `os.environ[...]`, `%VAR%`.
- Names without a value (`"token": ""`, `password = null`).

Keep a hit when the value looks like a real secret, **or when you cannot tell** — when in doubt, report it as "perlu dicek".

## 5. Git history (only if the repo has commits)

Values removed from the working tree can still live in history. Run `git log --all --format=%h -G'<token-format patterns from the table, POSIX ERE>'` and triage any commits that come back the same way. Skip this step if there is no history.

## 6. Report (Bahasa Indonesia)

**Never print a secret value.** Show at most the first 4 characters then `****` (e.g. `ghp_****`).

If something real was found:

```
✘ JANGAN COMMIT / PUSH — ditemukan <N> kredensial
- <file>:<baris> — <jenis, mis. GitHub PAT> — <ghp_****> [tracked | untracked | riwayat commit <hash>]
```

then list what the user must do: remove the value, move it to an environment variable or an ignored local file, and **revoke/rotate** it — a value that was ever committed stays in git history, so removing the line is not enough. Items marked "perlu dicek" go in a separate list. End by telling the user to run `/check-secrets` again after cleaning up.

If nothing was found:

```
✔ Aman untuk di-commit/push — <N> file diperiksa, tidak ada kredensial sungguhan.
```

State how many hits were dismissed as false positives and why (one line), so the user can see the check actually ran. If a step could not run (not a git repo, no history), say so rather than implying it passed.
