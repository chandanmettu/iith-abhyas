# Abhyas

**Academic resources, reimagined** — IIT Hyderabad's searchable archive of
notes, assignments, papers and reference-book pointers.

| | |
|---|---|
| **Live** | [abhyas.iith.online](https://abhyas.iith.online) |
| **Repository** | `github.com/chandanmettu/iith-resource-hub` (public). The local folder is `IITH Resource Hub`, which was the old name. |
| **Push via** | SSH host alias `github-iith-resource-hub` (deploy key `~/.ssh/iith-resource-hub-deploy`) |
| **Deploy** | Hostinger Git auto-deploy from `main`. **A push is a production release.** |
| **Agent policy** | Commit when asked and **ask before pushing** (no standing auto-push). |
| **Stack** | Plain HTML/CSS/JS plus a small PHP/JSON backend. No build step, no database. |

Abhyas is a live service, not the old placeholder prototype described in some
historical planning documents. On 2026-09-05, the production API exposed 864
published resources and 20 contributors. Treat that as a dated snapshot; the
server-managed catalogue changes independently of Git.

## What is live

- Archive search/filtering across course, branch, year and resource type
- Bookshelf for reference-book records without hosting copyrighted book PDFs
- Honor Roll derived from real published contributions
- resource detail pages and approved PDF delivery
- public contribution form with PDF validation, cooldown and duplicate checks
- quarantine outside the web root for unreviewed submissions
- authenticated admin console for preview, correction, approval/rejection,
  direct publishing, editing and catalogue removal
- atomic JSON writes plus rotating metadata backups
- Releases, Terms and project context pages

Public submissions never become public automatically. An admin must open and
approve each item; rejected and pending files stay outside `public_html`.

## Architecture

```text
HTML/CSS/JS pages          tracked in Git and served from public_html
api/data.php               read seam for live public JSON
api/submit.php             validates and quarantines public submissions
api/publish.php            authenticated publishing/moderation actions
admin/                     review and management console
files/                     approved public PDFs/covers
docs/history/              superseded planning docs (why: no DB, no WordPress, quarantine)
_local/                    git-ignored: lab pages, migration scripts, design refs,
                           a backup copy of the server config (password hash only)

On the server, ABOVE public_html (never in this repo):
abhyas-private/            config.php (admin password hash, paths) + metadata backups
abhyas-pending/            pending/rejected public submissions
```

Those two server directories are operational state and must never be added
to this repository. The mutable production JSON and uploaded documents also
need backups independent of both the hosting account and this laptop.

## Local preview

PHP is not installed on the dev Mac. Static pages render locally, but the
archive comes up empty and the admin console can't sign in. Check anything
data-driven against the live site.

Use HTTP because the data layer uses `fetch()`:

```sh
python3 -m http.server 8000
```

Open `http://localhost:8000`. The tracked JSON files support a local snapshot;
they are not automatically the current production catalogue.

## Maintaining and deploying

1. Read [`DESIGN.md`](DESIGN.md) before visual work.
2. Read [`ADMIN.md`](ADMIN.md) before publishing or moderating resources.
3. Read [`HANDOVER.md`](HANDOVER.md) for ownership, backup and recovery.
4. Review the workspace [`DEPLOY.md`](../DEPLOY.md) before a Git release.
5. Bump every affected CSS/JS `?v=` reference after edits, then run local
   checks and `git diff --check` before pushing.
6. Verify the live page/API and any removed server file with a cache-busted
   request after Hostinger deploys.

The PHP login/session/throttle/CSRF layer is active. The outer Basic Auth lines
in `admin/.htaccess` remain disabled and should be enabled after confirming the
host path and maintaining a recoverable second-admin setup.

## Current priorities

1. Enable and verify the outer `/admin/` Basic Auth layer.
2. Establish scheduled, off-host backups for uploaded PDFs and mutable JSON.
3. Replace the student-specific takedown contact with a durable project address.
4. Document at least two maintainers for hosting, domain, repository and any
   supporting script ownership.
5. Historical planning lives in [`docs/history/`](docs/history/). It explains
   the design, but this README and `HANDOVER.md` describe how things are now.
