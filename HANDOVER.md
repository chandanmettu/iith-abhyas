# Abhyas — handover

If you have just taken this over, read this file first. It covers what the
site is made of, which accounts it depends on, and what breaks if one of them
lapses.

`DESIGN.md` covers how it looks and why. This file covers how it stays alive.

---

## 1. The 60-second version

Abhyas is a **static-first website with a small PHP/JSON backend**. HTML, CSS
and JavaScript still have no build step, while PHP handles public submission,
quarantine, moderation and authenticated publishing. Production data is read
through `api/data.php`; serve the project over HTTP for a representative local
preview.

That is deliberate, and it keeps recovery simple: the public frontend can be
served by any static host, while full submission/moderation recovery requires a
PHP host plus the server-only config, JSON and uploaded-file backups described
below. There is no database migration.

| Page | File |
|---|---|
| Archive | `index.html` |
| Bookshelf | `library.html` |
| Honor Roll | `leaderboard.html` |
| Contribute | `contribute.html` |
| Releases | `releases.html` |
| Terms | `terms.html` |
| One resource's own page | `resource.html?id=N` |

All styling is in **one** file, `styles.css`. `data.js` contains configuration;
resource/contributor records are JSON served through the data API.

**Cache-busting is manual.** Every `<link>` and `<script>` tag carries a
`?v=N`. If you edit a file and the change does not appear in the browser,
you forgot to increase that number.

---

## 2. Accounts this depends on

Fill this in and keep it current. **At least two people should hold access to
every row.** The most common way a student project dies is a renewal notice
going to an inbox nobody reads any more.

| What | Provider | Registered to | Renews | Who has access |
|---|---|---|---|---|
| Domain (`iith.online`) | | | | |
| Web hosting | Hostinger | | | |
| Code repository | GitHub | `chandanmettu` | n/a | |
| Vote counter API | Google Apps Script | | n/a | |
| PDF viewer | Self-hosted PDF.js | n/a | n/a | repository |

---

## 3. Known single points of failure

These are **known and were deliberately deferred**, not overlooked. Each one
is quick to fix; the entry tells you how.

### 3.1 The contact address is a student email

`ms24btech11021@iith.ac.in` is hardcoded in **11 places** across all six
pages. It is a roll-number address, so it will be deactivated after that
student graduates.

This matters more than it looks: **`terms.html` promises that takedown
requests go to this inbox.** A takedown promise pointing at a dead mailbox is
the kind of thing that becomes somebody's problem.

To change it everywhere, from the project folder:

```bash
grep -rl 'ms24btech11021@iith.ac.in' *.html | xargs sed -i '' 's/ms24btech11021@iith\.ac\.in/NEW@ADDRESS/g'
```

Then check nothing was missed:

```bash
grep -c 'ms24btech11021@iith.ac.in' *.html
```

**Best fix:** an address on the domain the project already owns
(`abhyas@iith.online`) forwarded to whoever currently maintains the site.
Then the address on the site never has to change again — only the forwarding
target does.

### 3.2 The vote counter runs on a personal Google account

`releases.js` calls a Google Apps Script web app for the release vote count.
Apps Script web apps deployed as *"Execute as: Me"* stop working when that
Google account is deactivated, and they fail **silently** — the number just
stops moving.

The script source is kept in `_local/votes-apps-script.gs`, so it can be
redeployed under a different account. Or delete the counter; it is a
nice-to-have, and it currently costs a whole account dependency.

### 3.3 The code is on a personal GitHub account

The repository sits under a personal account rather than an organisation.
Personal repos cannot have co-owners, so transferring it requires that
person to still have access when the time comes.

Moving it to a GitHub **organisation** takes a few minutes, keeps the full
history, and lets several people be owners at once.

---

## 4. Back up the PDFs. This is the most important item here.

Everything else in this project is replaceable. The code is in a repository,
the design is documented, the layout can be rebuilt.

**The collected past papers cannot be replaced.** If a hosting renewal lapses
and those files existed only in that account, years of contributions are gone
permanently — and that archive is the entire point of the project.

- Keep a periodic zip of the uploaded files somewhere that is **neither** the
  hosting account **nor** one student's laptop.
- A copy held by the department, or by two maintainers, is enough.
- Do this before worrying about anything else on this list.

---

## 5. How resources work today

Resource data lives in **JSON**, not JavaScript:

| File | Holds |
|---|---|
| `resources.json` | every resource in the archive |
| `contributors.json` | who shared what, keyed by id |
| `courses.json` | course-code registry (used by the review console) |
| `data.js` | **configuration only** — branches, point values, semester dates |

`fetchResources()` in `data.js` is the single seam between the site and its
data. Pages wait on `ABHYAS_READY` before their first paint.

**Serving locally:** `fetch()` does not work over `file://`, so opening
`index.html` by double-clicking will show a "could not load" notice. Serve the
folder instead:

```bash
python3 -m http.server 8000
```

The read-only frontend can be re-hosted on any web server. Public submissions
and admin moderation additionally require PHP and the private directories.

**Production is populated.** On 2026-09-05 the live API exposed 864 resources
and 20 contributors. The tracked JSON is a development snapshot; mutable live
JSON and uploaded files are server-managed and can move ahead of Git.

---

## 5a. The backend

The PHP layer now covers both authenticated publishing and public submissions.
`docs/history/BACKEND-PLAN-v3.md` records the earlier admin-only phase; its Phase 4 section
documents the additive quarantine/moderation build.

| Path | Job |
|---|---|
| `api/data.php` | exposes the approved public JSON safely |
| `api/submit.php` | validates and writes public submissions to quarantine |
| `api/publish.php` | authenticated publish, review, approve/reject, edit and delete actions |
| `api/config.sample.php` | template — copy it OUTSIDE `public_html` |
| `admin/` | the console — publish form + manage list |

Public submission, quarantine and the moderation queue are deployed. Pending
files live outside the web root and are streamed only through authenticated
review actions. See `docs/history/BACKEND-PLAN-v3.md` §6.

### Setup, once

1. Create both `abhyas-private/` and `abhyas-pending/` **above** `public_html`.
2. Copy `api/config.sample.php` to `abhyas-private/config.php` and edit paths.
3. Open `api/hash.php`, generate a password hash, paste it into the config,
   then **delete `api/hash.php`**.
4. Uncomment the Basic Auth lines in `admin/.htaccess`.
5. Verify `api/data.php`, public submission, pending preview, approval,
   rejection, editing and logout before opening the workflow to contributors.

### Two rules that are not optional

1. **`admin/` is protected at the server, not just in the page.** The
   JavaScript that hides the console is convenience, not a security boundary
   — `.htaccess` Basic Auth in front, a PHP session re-checked on every
   request behind it. Neither layer substitutes for the other.
2. **Every state-changing request needs a valid CSRF token**, issued at
   login. A session cookie alone doesn't stop a forged request from
   another tab — the token is what proves a request actually came from
   this console.

### Status

The PHP backend and console are live. The application-level login, session,
throttling and CSRF checks are active. The outer Basic Auth lines in
`admin/.htaccess` are still commented and remain the main hardening item.

---

## 6. Editing rules worth keeping

- **Branch and resource type must stay dropdowns, never free text.** If one
  person types `CSE` and the next types `Computer Science`, the filters
  quietly stop working and nobody notices for months.
- `DEPARTMENTS` in `data.js` is the single source of truth for the fifteen
  branches. The Bookshelf's shelves, the filter pills and the Contribute page
  all read from it. Never hardcode a branch list anywhere else.
- Read `DESIGN.md` before changing any CSS. It is a contract, and the
  reasoning behind each rule is written down.

---

## 7. Putting the site back up from nothing

1. Get the files (repository, or a backup copy).
2. Upload the frontend to a web host; use a PHP-capable host for the full
   submission/admin service.
3. Point the domain at it.
4. Re-upload `/files/` from the backup in §4.

There is no database, but there is server state to restore: private config,
mutable JSON, approved uploads, pending/rejected submissions and backups. Keep
an off-host copy of the uploaded files; Git alone cannot recreate the service.
