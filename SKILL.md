---
name: 100mini-upload
description: Use when publishing, sharing, uploading, or hosting an HTML page (single-file HTML or a static site as HTML/CSS/JS or .zip) as a shareable link, or when the user asks to "post to 100mini" / "share link" / "upload to 100mini" / "publish learning page". Use to upload from the command line without a browser or login.
---

# 100mini Upload

Publish HTML pages to 100mini (https://www.100mini.com) as a shareable link. Two modes:
- **Anonymous**: no login, no key. 5 uploads/day/IP, pages auto-destroy after 7 days.
- **Authenticated** (`--token`): permanent pages, no daily cap, full metadata (title/tags/share-to-square), subject to the same points policy as the web UI.

## When to Use

- User asks to publish/share/upload an HTML page or make a shareable link for one
- You built an HTML page (learning page) and need a URL to share
- User wants to "upload to 100mini", "publish learning page", "share link", "post to 100mini"
- Any script or automation needs temporary static hosting
- Agent needs to upload pages on behalf of a logged-in user (use `--token`)

When NOT to use: page must persist longer than 7 days or be managed (delete/edit/track views) — use authenticated mode with a token.

## Install

1. Copy the script into your project:
   ```bash
   cp <skill_dir>/scripts/upload.mjs scripts/upload.mjs
   ```
2. No dependencies. Requires Node.js ≥ 18 (uses built-in `fetch`, `FormData`, `Blob`).

## Usage

```bash
# --- Anonymous mode (default) ---
node scripts/upload.mjs --file page.html
node scripts/upload.mjs --content "<h1>hi</h1>"
cat page.html | node scripts/upload.mjs

# --- Authenticated mode (permanent pages) ---
node scripts/upload.mjs --token 100m_xxxxxxxxxxxxxxxx --file page.html
node scripts/upload.mjs --token 100m_xxxx --file app.zip --title "My App"

# --- Env var fallback ---
export MINI_TOKEN=100m_xxx
node scripts/upload.mjs --file page.html        # uses env token
node scripts/upload.mjs --anonymous --file page.html  # force anonymous even if env set

# --- Common ---
node scripts/upload.mjs --file page.html --quiet  # print URL only
```

### Options

| Option | Description | Default |
|---|---|---|
| `--file <path>` | `.html`/`.htm`/`.zip` file path (zip assets uploaded as a site) | — |
| `--content <html>` | HTML string | — |
| `--title <title>` | page title (required for authenticated uploads) | — |
| `--tags <a,b>` | comma-separated tags | — |
| `--category <cat>` | category | `general` |
| `--base-url <url>` | upload target | `https://www.100mini.com` |
| `--token <key>` | API key for authenticated upload (see below) | — |
| `--anonymous` | force anonymous mode even if `MINI_TOKEN` env is set | `false` |
| `--quiet` | print only the final URL | `false` |
| `-h, --help` | show help | — |

Provide exactly one of `--file`, `--content`, or stdin.

### Authenticated mode rules

- `--title` is **required** for authenticated uploads (server rejects blank titles).
- Pages are **permanent** (no 7-day expiry).
- No daily upload cap (anonymous limit is 5/day/IP).
- Points policy: same as the web UI — free permanent-page quota (default 5 + bonus) applies; beyond that each upload costs 10 points. Active members are exempt from points. See server logic in `pages.service.ts`.

## Getting an API Token

Tokens are generated via the 100mini web UI (no CLI login):

1. Log in at https://www.100mini.com
2. Go to **My Pages** (`/links`)
3. Open the **API Keys** section
4. Enter a name (e.g. "cli-agent") → click **Generate**
5. Copy the key (`100m_...`) — it is shown **once only**

### Using the token

Option A — pass directly:
```bash
node scripts/upload.mjs --token 100m_xxxxxx --file page.html
```

Option B — set as env var (recommended for repeated use):
```bash
export MINI_TOKEN=100m_xxxxxx
node scripts/upload.mjs --file page.html
```

Option C — save in a `.env` file and source before running:
```bash
echo 'MINI_TOKEN=100m_xxxxxx' >> .env
source .env && node scripts/upload.mjs --file page.html
```

The token can be revoked at any time from the web UI. Once revoked, authenticated uploads fail with exit 1 and a server error message.

## Output

Success — exit 0, JSON (or URL only with `--quiet`):

```json
{
  "id": "AbC1234",
  "url": "/p/AbC1234",
  "expiresAt": null,
  "isPermanent": true,
  "title": "My Page",
  "isSharedToSquare": false,
  "previewPath": null
}
```

Authenticated mode: `expiresAt` is `null` and `isPermanent` is `true`.
Anonymous mode: `expiresAt` is a future ISO timestamp and `isPermanent` is `false`.

Final shareable page: `https://www.100mini.com/p/<id>` (or `--base-url`).

Failures — errors on stderr:
- exit 2: bad arguments (no input, wrong extension, >5MB)
- exit 1: upload failed (server rejection, network error, invalid/expired token)

## Common Mistakes

- **No input:** must pass `--file`, `--content`, or pipe stdin, otherwise exit 2.
- **ZIP needs an entry file:** the archive must contain `index.html` (or another `.html` at its root).
- **5MB limit:** single file or unzipped total must stay under 5MB.
- **Anonymous 7-day expiry:** anonymous pages are deleted automatically; use `--token` for permanent hosting.
- **Missing title:** authenticated uploads require `--title`; blank title returns exit 1 with "title is required".
- **Invalid token:** returns exit 1 with `Invalid or expired API key`; check token hasn't been revoked.
- **Points exhausted:** if beyond free quota and points < 10, returns exit 1 with "insufficient points"; top up via the web UI.
- **Server error messages may be in Chinese** (e.g. "only .html or .zip files supported", "anonymous upload limit reached"); relay them to the user verbatim or paraphrase.

## Rate Limits

| Mode | Daily limit | Storage | Points |
|---|---|---|---|
| Anonymous | 5 uploads/IP/day | 7-day temp (`tmp/` prefix) | — |
| Authenticated | unlimited | permanent | subject to web policy (see above) |


