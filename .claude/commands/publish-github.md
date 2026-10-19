---
description: Security-scan, push to GitHub, write the README and About section, and deploy GitHub Pages with CI/CD
argument-hint: <owner/repo or repo URL> [public|private] [branch]
allowed-tools: Bash, PowerShell, Read, Write, Edit, Grep, Glob
---

# Publish this project to GitHub

GitHub info from the user: `$ARGUMENTS`

Parse it as: repository (`owner/repo` or a `https://github.com/owner/repo(.git)` URL), optional visibility (`public` default, `private`), optional branch (default `main`). If no repository was given, use the existing `origin` remote; if there is none, stop and ask the user for it. Never guess an owner name.

Follow the steps **in order**. Step 1 is a gate: if it fails, stop, report the findings and push nothing. Use the `gh` CLI for GitHub; never ask for, print, or write a token or password. Before every outward step (creating the repo, pushing, changing settings) state what you are about to do in one line.

## 0. Preflight

- `git status`, `git remote -v`, `git branch --show-current`, `gh auth status`. If `gh` is missing or not logged in, stop and tell the user to run `gh auth login` themselves.
- Read `CLAUDE.md` for the project's constraints and what the app is.
- Check whether the repo exists: `gh repo view <owner/repo> --json name,visibility,homepageUrl,description,repositoryTopics`.

## 1. Security scan (gate — nothing leaves the machine before this passes)

Scan **every file that would be pushed** (`git ls-files` plus untracked, not-ignored files from `git ls-files --others --exclude-standard`) and **the full git history** (`git log -p --all`), not just the working tree.

1. **Secrets / credentials** — search for: private keys (`-----BEGIN .*PRIVATE KEY-----`), AWS keys (`AKIA[0-9A-Z]{16}`), GitHub tokens (`gh[pousr]_[A-Za-z0-9]{36,}`, `github_pat_`), Slack tokens (`xox[baprs]-`), Google keys (`AIza[0-9A-Za-z_-]{35}`), generic `(api[_-]?key|secret|token|passw(or)?d)\s*[:=]\s*['"][^'"]{8,}`, connection strings with embedded credentials (`://[^/\s:]+:[^@\s]+@`). If `gitleaks` or `trufflehog` is installed, also run it (`gitleaks detect --source . --log-opts=--all`); do not install anything new without asking.
2. **Files that must not be published** — `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.sqlite`/`*.db`, credential JSON, `.claude/settings.local.json`, OS/editor junk, anything large (>5 MB). Make sure `.gitignore` covers them (add entries if needed); if one is already tracked, `git rm --cached` it and tell the user. If a secret is in **history**, stop: tell the user to rotate it and rewrite history themselves.
3. **Personal data** — real email addresses, phone numbers, names or account numbers beyond the intended demo data. The only email expected is the one in the `FORMSUBMIT_ENDPOINT` constant; flag any other and ask before pushing. Flag the user's own email if it appears anywhere.
4. **App does not send data to the internet** — the app must make no network calls except the one the spec allows. Run:
   ```bash
   rg -n "localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important|https?://" index.html
   rg -n "fetch\(|XMLHttpRequest|sendBeacon|WebSocket|EventSource|navigator\.sendBeacon|<script[^>]+src=|<link[^>]+href=\"https?:|<img[^>]+src=\"https?:|@import|url\(https?:" index.html
   ```
   The only allowed hit is the `FORMSUBMIT_ENDPOINT` line and the single `fetch` inside `notifyNewTask()`. Any CDN script, web font, analytics, tracker, external image or second endpoint is a failure: report it and stop.
5. **Workflow safety** — `.github/workflows/*.yml` must use least-privilege `permissions:`, no `pull_request_target`, no secrets echoed, and only official `actions/*` actions pinned to a major version.

Print a short table: check → pass/fail → details (file:line). Continue only if everything passes, or the user explicitly accepts a specific finding.

## 2. Push the code

- If the repo does not exist: `gh repo create <owner/repo> --<visibility> --source . --remote origin` (no `--push` yet). GitHub Pages on a private repo needs a paid plan — warn if `private`.
- If `origin` is missing or points elsewhere, set it to `https://github.com/<owner/repo>.git` (ask before replacing a different existing URL).
- Make sure the branch is the requested one (`git branch -M <branch>` only if the repo has no other branches in use).
- Commit any pending work with a clear message, then `git push -u origin <branch>`. Never `--force` unless the user says so.

## 3. README.md (create or update)

Read `index.html` and `CLAUDE.md` first and describe the real app; do not invent features. If a README exists, keep any user-written sections and update the rest. Include:

- Title and one-paragraph summary (internal demo Kanban board for a fictitious bank's IT PMO; no real bank branding).
- **Live demo** link: `https://<owner>.github.io/<repo>/` (lowercase owner).
- **Tech badges** (shields.io, static, no tracking params), e.g.:
  `![HTML5](https://img.shields.io/badge/HTML5-E34F26?logo=html5&logoColor=white)`
  `![CSS3](https://img.shields.io/badge/CSS3-1572B6?logo=css3&logoColor=white)`
  `![JavaScript](https://img.shields.io/badge/JavaScript-Vanilla-F7DF1E?logo=javascript&logoColor=black)`
  `![No dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)`
  plus the workflow status badge `https://github.com/<owner>/<repo>/actions/workflows/pages.yml/badge.svg` and a License badge only if a LICENSE file exists.
- **Features** (columns, drag-and-drop and Move menu, filters, overdue highlighting, add-task modal with validation, email notification via FormSubmit, keyboard accessible).
- **How to install** — `git clone https://github.com/<owner>/<repo>.git`; no build, npm or dependencies.
- **How to run** — double-click `index.html` (works from `file://`), or `python -m http.server 8000` and open `http://localhost:8000`, or use the live GitHub Pages link.
- **Configuration** — FormSubmit: set `FORMSUBMIT_ENDPOINT` once, one-time activation email, and that a failure only shows a warning toast.
- **Notes** — no persistence by design (refresh resets to seed data); the only network call is FormSubmit.
- **Deployment / CI** — what the workflow does.
- **Project structure** — short file list.

## 4. CI/CD (GitHub Actions)

Create or update `.github/workflows/pages.yml` so that on push to `<branch>`, pull requests, and `workflow_dispatch`:

- A **`check`** job (`permissions: contents: read`) that fails the build when the constraint grep from step 1.4 finds anything other than the `FORMSUBMIT_ENDPOINT` line / its `fetch`, when a secret pattern from step 1.1 matches, or when `index.html` is missing. Keep it plain shell (`grep -nE`) — no third-party actions.
- A **`deploy`** job (`needs: check`, only on push/dispatch, not PRs) with `pages: write`, `id-token: write`, the `github-pages` environment, and `actions/checkout@v4` → copy only `index.html` (plus `.nojekyll`) into `_site` → `actions/configure-pages@v5` → `actions/upload-pages-artifact@v3` → `actions/deploy-pages@v4`. Do not publish `CLAUDE.md`, `.claude/` or other repo files.
- Keep `concurrency: group: pages`.

Commit README + workflow and push.

## 5. GitHub Pages

- Enable Pages with the Actions source:
  `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow` (if it already exists: `gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow`).
- Watch the run: `gh run list --workflow pages.yml --limit 1`, then `gh run watch <id> --exit-status` (run it in the background). If it fails, read `gh run view <id> --log-failed`, fix, push, and retry.
- Confirm the URL: `gh api repos/<owner>/<repo>/pages --jq .html_url`.

## 6. About section

`gh repo edit <owner/repo> --description "<one line, ≤ 120 chars, no real bank name>" --homepage "<pages URL>" --add-topic kanban --add-topic project-management --add-topic html --add-topic css --add-topic javascript --add-topic vanilla-js --add-topic github-pages`

Then verify with `gh repo view <owner/repo> --json description,homepageUrl,repositoryTopics` that the homepage is the Pages link.

## 7. Final security check and report

- Re-run step 1 against what is actually on GitHub: `git fetch origin && git diff origin/<branch> --stat` should be empty, and re-scan `git ls-files`.
- Fetch the live page (`curl -sI <pages URL>`) and confirm HTTP 200.
- Report: repo URL, Pages URL, workflow run result, files pushed, the security-scan table, and anything skipped or needing the user's action (e.g. FormSubmit activation, secret rotation).
