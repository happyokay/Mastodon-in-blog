# Mastodon in Blog

Embed a Mastodon-powered microblog stream inside WordPress, Bear Blog, or any page that can load a small script.

This project is based on [MyStatus](https://github.com/verfasor/MyStatus). It keeps the original single-file Cloudflare Worker shape, D1 storage, admin UI, Atom feed, sitemap, and `/client.js` embed script, then adds Mastodon sync.

## What It Does

- Syncs your latest Mastodon statuses into Cloudflare D1.
- Stores toot text, timestamps, original toot URLs, and remote media URLs.
- Does not store image/video binaries in D1.
- Keeps imported statuses deduplicated by Mastodon status ID.
- Serves a public status page from the Worker.
- Exposes `/client.js` so your stream can be embedded inside WordPress or Bear Blog.
- Supports Cloudflare Cron triggers and a protected manual sync API.

## Required Cloudflare Bindings

Create a D1 database and bind it to the Worker as:

```text
DB
```

The Worker creates and migrates the required tables on first request.

## Required Environment Variables

```text
ADMIN_PASSWORD=change-me
SESSION_SECRET=long-random-secret
MASTODON_INSTANCE_URL=https://mastodon.social
MASTODON_USERNAME=yourname
ALLOWED_ORIGINS=https://your-bearblog.example,https://your-wordpress.example
```

You can use `MASTODON_ACCOUNT_ID` instead of `MASTODON_USERNAME` if you already know the account ID.

For private, followers-only, or otherwise token-gated statuses, also set:

```text
MASTODON_ACCESS_TOKEN=your-token
```

## Optional Environment Variables

```text
MASTODON_IMPORT_LIMIT=20
MASTODON_INCLUDE_REPLIES=false
MASTODON_INCLUDE_REBLOGS=false
SITENAME=Status
SITE_DESCRIPTION=A personal status stream.
API_URL=https://your-worker.example.workers.dev
```

`MASTODON_IMPORT_LIMIT` is capped at 40 per sync run.

## Deploy

Use `wrangler.toml.example` as a starting point:

```bash
cp wrangler.toml.example wrangler.toml
```

Fill in your Worker name, D1 database ID, blog origins, and Mastodon instance/account values.

Set secrets:

```bash
wrangler secret put ADMIN_PASSWORD
wrangler secret put SESSION_SECRET
```

If you need a Mastodon token:

```bash
wrangler secret put MASTODON_ACCESS_TOKEN
```

Deploy:

```bash
wrangler deploy
```

## Manual Sync

After logging in at `/login`, you can manually trigger a sync with:

```bash
curl -X POST https://your-worker.example.workers.dev/api/sync/mastodon \
  -H "Origin: https://your-worker.example.workers.dev" \
  -H "Cookie: gb_session=..."
```

In normal use, let the Cron trigger run it automatically.

## Embed In Bear Blog

Create a page such as `/status/` and add:

```html
<div data-gb data-gb-api-url="https://your-worker.example.workers.dev"></div>
<script src="https://your-worker.example.workers.dev/client.js"></script>
```

Add your Bear Blog origin to `ALLOWED_ORIGINS`.

## Embed In WordPress

For self-hosted WordPress, add a Custom HTML block:

```html
<div data-gb data-gb-api-url="https://your-worker.example.workers.dev"></div>
<script src="https://your-worker.example.workers.dev/client.js"></script>
```

Some WordPress.com plans restrict custom JavaScript. If the script is stripped, use self-hosted WordPress or a WordPress plan/plugin that allows custom scripts.

Add your WordPress origin to `ALLOWED_ORIGINS`.

## Styling The Embedded Stream

The embed injects markup under your `[data-gb]` container. Add CSS in the host blog.

Useful classes:

| Class | Role |
| --- | --- |
| `.gb-widget` | Widget root |
| `.gb-entries-list` | List container |
| `.gb-entry` | One status block |
| `.gb-entry-content` | Rendered status content |
| `.gb-entry-meta` | Date/footer row |
| `.gb-load-more-btn` | Load more button |

Example:

```html
<style>
.gb-entry {
  border-bottom: 1px solid currentColor;
  padding: 1rem 0;
}
.gb-entry-content img {
  max-width: 100%;
  height: auto;
  border-radius: 8px;
}
.gb-entry-meta {
  opacity: .72;
  font-size: .9em;
}
</style>
```

## Data Notes

D1 stores text, timestamps, Mastodon IDs, original toot links, and media URLs. Images remain hosted by the Mastodon instance/CDN unless you separately add an R2 media archive flow.

## License

GNU AGPL v3, following the upstream MyStatus license.
