   # WordPress (XAMPP) — Team Setup via GitHub

Collaborate on a full WordPress project using Git + GitHub for **code**, and shared snapshots for **database** and **uploads**.

> **Repo layout:** The Git repo root is the **`wordpress/`** folder inside XAMPP’s `htdocs`.

- **Windows XAMPP path:** `C:\xampp\htdocs\wordpress`
- **macOS/Linux (MAMP/LAMP-ish):** `~/Sites/wordpress` (adjust as needed)

---

## 0) Prerequisites

- **XAMPP** (Apache + MySQL + PHP)
- **Git** (use Git Bash on Windows, or Terminal on macOS/Linux)
- **WP-CLI** (recommended) → optional; phpMyAdmin steps included as alternatives
- A private **GitHub** repo you control (or create one during step 2)

---

## 1) Repository Structure

Everything is committed **except** secrets and heavy media:

```
wordpress/            ← 🧷 Git repo root
├─ wp-admin/
├─ wp-includes/
├─ wp-content/
│  ├─ themes/
│  ├─ plugins/
│  └─ uploads/        (🚫 ignored from Git)
├─ wp-config-sample.php   (✅ commit)
├─ wp-config.php          (🚫 ignored; each dev keeps their own)
└─ ...
```

### `.gitignore` (place in `wordpress/`)

```gitignore
# Local config & secrets
wp-config.php
.env
.env.*

# DB dumps
*.sql
*.sql.gz

# Media & caches
wp-content/uploads/
wp-content/cache/
wp-content/upgrade/

# Build artifacts & OS clutter
node_modules/
vendor/
dist/
*.log
.DS_Store
Thumbs.db
```

---

## 2) Initialize the GitHub Repo (project owner)

From `htdocs/wordpress`:

```bash
git init
git branch -m main
git add -A
git commit -m "Initial commit: full WP project (uploads & secrets ignored)"
git remote add origin <YOUR-GITHUB-REPO-URL>
git push -u origin main
```

Invite your two collaborators to the repo (**GitHub → Settings → Collaborators**).

---

## 3) Configure `wp-config.php` (each developer)

Each dev keeps a **local** `wp-config.php` (not committed). Create it by copying `wp-config-sample.php`, fill DB creds, and add the following lines **above** “That’s all, stop editing!”:

```php
// Flexible site URLs for any local path (e.g., /wordpress or a subfolder)
$scheme = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https://' : 'http://';
$host   = $_SERVER['HTTP_HOST'] ?? 'localhost';
$base   = rtrim(dirname($_SERVER['PHP_SELF']), '/\'); // handles /wordpress subfolder
define('WP_HOME',    $scheme . $host . $base);
define('WP_SITEURL', $scheme . $host . $base);
```

This avoids hardcoding URLs and prevents login loops when folder paths differ.

> Optionally load secrets from `.env` (never commit `.env`).

---

## 4) Create & Share the Seed Database (project owner, one-time)

**With WP-CLI (recommended):**

```bash
# run inside wordpress/
wp db export seed.sql
```

**Without WP-CLI (phpMyAdmin):**
- Open `http://localhost/phpmyadmin`
- Select your WordPress DB → **Export** → Quick → SQL → Download as `seed.sql`

Share `seed.sql` with collaborators via Drive/Dropbox (don’t commit it).  
If you already have media, zip `wp-content/uploads/` and share that once too.

---

## 5) Collaborators: First-Time Setup

1. Install **XAMPP**. Start Apache & MySQL.
2. Create a **new empty database** (e.g., `wp_dev`) via phpMyAdmin.
3. **Clone the repo** into your XAMPP web root:

   **Windows (Git Bash):**
   ```bash
   cd /c/xampp/htdocs
   git clone <YOUR-GITHUB-REPO-URL> wordpress
   ```

   **macOS/Linux (example path):**
   ```bash
   cd ~/Sites
   git clone <YOUR-GITHUB-REPO-URL> wordpress
   ```

4. Copy `wp-config-sample.php` → `wp-config.php`, put your DB creds, and add the flexible URL lines from step 3.
5. **Import the DB:**

   **WP-CLI:**
   ```bash
   cd wordpress
   wp db import /path/to/seed.sql
   ```

   **phpMyAdmin:** Select your DB → **Import** → choose `seed.sql`.

6. (Optional) Unzip the shared `uploads/` into `wp-content/uploads/`.
7. Visit `http://localhost/wordpress` and log in.

> **If permalinks 404:** Admin → Settings → Permalinks → **Save**.

---

## 6) Daily Workflow

- **Create a branch**: `git switch -c feature/something`
- **Code changes** in `themes/` or `plugins/`
- **Commit & push**; open a PR on GitHub
- **Review & merge** (main stays deployable)
- **Pull latest** on your local: `git pull`

> **Do not commit** `uploads/`, DB dumps, or `wp-config.php`.

---

## 7) Sharing Database Changes (users, settings, CPTs)

When someone makes DB-level changes others need (new users, plugin options, CPT registration side effects):

**Exporter does:**

```bash
wp db export snap-YYYYMMDD.sql
```

Share the SQL file (not in Git).

**Others do:**

```bash
wp db import snap-YYYYMMDD.sql
# Safe search/replace (handles serialized data). Adjust URLs if your path differs.
wp search-replace 'http://localhost/wordpress' 'http://localhost/wordpress' --skip-columns=guid
```

If paths are identical, the search-replace is a no-op but safe to run.

**Alternative:** Create local users with WP-CLI instead of sharing a DB:

```bash
wp user create alice alice@example.com --role=editor --user_pass='TempPass123!'
wp user create bob   bob@example.com   --role=editor --user_pass='TempPass123!'
```

---

## 8) Media Strategy (uploads)

**Recommended:** keep `wp-content/uploads/` **out of Git**.

Options:
- **Occasional zip + share** (simple)
- **Rsync/WinSCP sync** (fast, incremental)
- **Offload plugin to object storage** (S3/R2/Spaces) if media grows
- **Git LFS** (last resort; quotas & history bloat still apply)

If you must track a **tiny shared subset**, allowlist a subfolder:

```gitignore
wp-content/uploads/*
!wp-content/uploads/.gitkeep
!wp-content/uploads/shared/**
```

---

## 9) Installing New Plugins (two supported workflows)

### A) Commit plugin code (simple, works now)

On a branch, install & test locally:

**WP-CLI:**

```bash
wp plugin install plugin-slug --version=1.2.3 --activate
```

*(Or upload via Admin → Plugins → Add New.)*

Commit the resulting folder under `wp-content/plugins/plugin-slug/`.

Push PR → review → merge.

Teammates pull and activate:

```bash
wp plugin activate plugin-slug
```

If the plugin adds required settings/tables, export & share a DB snapshot.

> **Premium plugins:** keep repo **private**; do not commit license keys. Store zips in a private location or use vendor’s updater. Put keys in `.env`/`wp-config.php` (not in Git).

### B) Composer-managed (optional, reproducible)

- Add Composer + wpackagist to `composer.json` and install plugins via Composer.
- **Pros:** reproducible installs; **Cons:** extra setup.
- Since this repo tracks the full project, **A** is perfectly fine.

---

## 10) Troubleshooting

- **Login loop / redirects:** confirm `WP_HOME` / `WP_SITEURL` lines exist in `wp-config.php`.  
  Check DB values: `wp option get home` and `wp option get siteurl`.
- **Permalinks 404:** Admin → Settings → Permalinks → **Save**.
- **Emails from localhost not sending:** use an SMTP plugin (for dev, set passwords directly).
- **Git EOL issues (Windows):**
  ```bash
  git config core.autocrlf true
  ```
- **Apache docroot mismatch:** ensure the project is at `http://localhost/wordpress` (or adjust path accordingly).

---

## 11) Team Conventions (copy/paste into PR template if you like)

- No committing `uploads/`, `wp-config.php`, `*.sql`, `.env`.
- Pin plugin versions in PR descriptions: `plugin-slug@1.2.3`.
- One feature per branch/PR; keep PRs small and reviewable.
- After merging, always `git pull` before starting new work.

---

## 12) (Optional) Staging Later

If you add a staging server:

- Code deploys from GitHub (rsync/SFTP/Actions).
- Use:
  ```bash
  wp search-replace 'http://localhost/wordpress' 'https://staging.example.com' --skip-columns=guid
  ```
  after DB import.
- Protect staging with HTTP auth; keep backups.

---

## Command Cheat Sheet

```bash
# Branching
git switch -c feature/my-change
git add -A && git commit -m "feat: my change"
git push -u origin feature/my-change

# Update local
git pull

# WP-CLI (run inside wordpress/)
wp db export seed.sql
wp db import seed.sql
wp search-replace 'http://localhost/wordpress' 'http://localhost/wordpress' --skip-columns=guid
wp user create alice alice@example.com --role=editor --user_pass='TempPass123!'
wp plugin install plugin-slug --version=1.2.3 --activate
wp plugin activate plugin-slug
```

## How to Test

1. `git fetch && git switch <this-branch>`
2. `git pull`                            

## Checklist

- [ ] Branched from `main` and rebased cleanly
- [ ] No `wp-config.php`, `.env`, `*.sql`, or `wp-content/uploads/` committed
- [ ] New plugins pinned (e.g., `plugin-slug@1.2.3`) and activation steps documented
- [ ] If DB changes required, attached/linked snapshot and instructions provided
- [ ] Built assets generated locally if needed (e.g., `npm run build`) and **not** committed if ignored
- [ ] Tested locally at `http://localhost/wordpress` (permalinks saved if needed)
- [ ] Considered security implications (nonces/escaping/cap checks for admin features)
- [ ] Updated docs (README/CONTRIBUTING) if behavior or setup changed

## Type of Change

- [ ] feat (new feature)
- [ ] fix (bug fix)
- [ ] chore (tooling/config)
- [ ] docs (documentation only)
- [ ] refactor (no functional change)
- [ ] perf (performance)

## Linked Issues

Closes #___
Refs #___
# Contributing

Thanks for helping build this WordPress project! This document explains how we work together and what a “good” contribution looks like.

> **TL;DR**  
> - Work on a branch → open a PR → request review → merge.  
> - Don’t commit `uploads/`, `wp-config.php`, `.env`, or `*.sql`.  
> - Pin plugin versions and document any DB changes other teammates must apply.

---

## Prerequisites
- XAMPP (Apache + MySQL + PHP)
- Git
- WP-CLI (recommended) — phpMyAdmin works too

Follow the **README.md** to set up your local environment.

---

## Branching & Commit Style

### Branch naming
- `feature/<short-name>` — new features
- `fix/<short-name>` — bug fixes
- `chore/<short-name>` — tooling, configs, cleanup
- `docs/<short-name>` — documentation only

Examples: `feature/hero-slider`, `fix/login-redirect`, `chore/ignore-uploads`

### Commit messages (Conventional Commits)
- `feat:` new feature
- `fix:` bug fix
- `chore:` tooling/config/no production code change
- `docs:` documentation only
- `refactor:` internal code change
- `style:` CSS/formatting (no logic)
- `perf:` performance improvement

Examples:
```
feat: add sticky header to theme
fix: correct nonce check on contact form
chore: ignore uploads and sql dumps
```

---

## What goes in Git (and what doesn’t)

**✅ Commit:**
- Theme and plugin **source code** (inside `wp-content/`)
- Build scripts and configuration (e.g., package.json, composer.json)
- Documentation (README, CONTRIBUTING)

**🚫 Do NOT commit:**
- `wp-config.php`, `.env*`
- Database dumps: `*.sql`, `*.sql.gz`
- Media: `wp-content/uploads/`
- Build artifacts: `dist/`, `vendor/`, `node_modules/` (unless intentionally vendor-committed)

> See the `.gitignore` in the repo root.

---

## Plugin Workflow

**Option A — Commit plugin code (simple):**
1. On a branch, install locally and test:
   ```bash
   wp plugin install plugin-slug --version=1.2.3 --activate
   ```
2. Commit the new plugin folder under `wp-content/plugins/plugin-slug/`.
3. Open a PR; in the description, **pin the version**: `plugin-slug@1.2.3`.
4. After merge, teammates pull and run:
   ```bash
   wp plugin activate plugin-slug
   ```
5. If the plugin requires settings/DB tables, **export a snapshot** and link it in the PR.

**Option B — Composer-managed (optional):**  
Use Composer + wpackagist to lock plugin versions. This is more reproducible but needs setup.

**Premium plugins:** keep the repo private, don’t commit license keys. Store zips/keys privately and load keys from `.env`/`wp-config.php`.

---

## Database & Media

- **DB snapshots:** When your change needs DB state (users, options, CPT settings), export and attach instructions:
  ```bash
  wp db export snap-YYYYMMDD.sql
  # teammates:
  wp db import snap-YYYYMMDD.sql
  wp search-replace 'http://localhost/wordpress' 'http://localhost/wordpress' --skip-columns=guid
  ```
- **Media (`uploads/`):** Do not commit. Share via zip/rsync/offload plugin.

---

## Code Style & Quality (lightweight)

- Follow WordPress Coding Standards when possible.
- Optional: run PHPCS if you have it set up (no requirement to block PRs).  
- Keep PRs scoped and reviewable (< ~300 lines changed when possible).

---

## Pull Request Process

1. Ensure your branch is up to date with `main` (rebase if needed).
2. Fill out the PR template (purpose, screenshots, test steps).
3. Request at least **one review**.
4. Address review comments, squash/rebase if requested.
5. Maintainer merges when CI/reviews pass.

---

## Security & Secrets

- Never commit secrets (API keys, SMTP creds, premium plugin keys).
- Keep secrets in `.env` / `wp-config.php` (untracked).

---

## Troubleshooting Quickies

- **Permalinks 404** → Admin → Settings → Permalinks → Save.
- **Login loop** → ensure `WP_HOME`/`WP_SITEURL` lines are in `wp-config.php`.
- **Emails don’t send** → use an SMTP plugin locally or set passwords directly.
- **Large files rejected by GitHub** → confirm `uploads/` and `*.sql` are ignored.

---

## Release / Staging (optional, later)

- If a staging server exists, deploy via GitHub Actions/rsync.  
- After DB import on staging:
  ```bash
  wp search-replace 'http://localhost/wordpress' 'https://staging.example.com' --skip-columns=guid
  ```

Thanks for contributing! 🎉
