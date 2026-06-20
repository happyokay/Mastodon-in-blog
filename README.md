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

The binding name must be exactly `DB`. The Worker creates and migrates the required tables automatically on the first request, so you do not need to run a separate schema initialization command.

## Required Environment Variables

```text
ADMIN_PASSWORD=change-me
SESSION_SECRET=long-random-secret
MASTODON_INSTANCE_URL=https://mastodon.social
MASTODON_USERNAME=yourname
ALLOWED_ORIGINS=https://yourname.bearblog.dev,https://your-wordpress.example
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

## Deploy From Mac Terminal

1. Install or confirm Node.js:

```bash
node --version
npm --version
```

If those commands are missing, install Node.js from [nodejs.org](https://nodejs.org/) or with Homebrew:

```bash
brew install node
```

2. Install Wrangler:

```bash
npm install -g wrangler
wrangler --version
```

3. Log in to Cloudflare:

```bash
wrangler login
```

4. Create the D1 database:

```bash
wrangler d1 create mastodon-in-blog
```

Copy the returned `database_id`. The D1 binding name in your config must remain `DB`.

5. Create your local Wrangler config:

Use `wrangler.toml.example` as a starting point:

```bash
cp wrangler.toml.example wrangler.toml
```

Edit `wrangler.toml` and fill in:

- `name`
- `database_id`
- `MASTODON_INSTANCE_URL`
- `MASTODON_USERNAME`
- `ALLOWED_ORIGINS`

For Bear Blog, `ALLOWED_ORIGINS` should contain the page origin only, such as `https://yourname.bearblog.dev` or your Bear Blog custom domain. If you also embed in WordPress, add that origin after a comma.

6. Set secrets:

```bash
wrangler secret put ADMIN_PASSWORD
wrangler secret put SESSION_SECRET
```

Use a long random value for `SESSION_SECRET`.

7. Optional: set a Mastodon access token.

If you need a Mastodon token:

```bash
wrangler secret put MASTODON_ACCESS_TOKEN
```

Public accounts can often sync without this token. Private, followers-only, or token-gated content needs it.

8. Deploy:

```bash
wrangler deploy
```

Open your Worker URL once after deployment. That first request initializes or migrates the D1 tables automatically.

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

Add your Bear Blog origin to `ALLOWED_ORIGINS`, for example `https://yourname.bearblog.dev` or your Bear Blog custom domain. Do not include a path like `/status/`.

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
