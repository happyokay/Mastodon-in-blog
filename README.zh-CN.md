# Mastodon in Blog

把 Mastodon 嘟文流嵌入 WordPress、Bear Blog，或任何可以加载小段脚本的页面。

本项目基于 [MyStatus](https://github.com/verfasor/MyStatus)。它保留了原项目单文件 Cloudflare Worker、D1 存储、管理后台、Atom feed、sitemap 和 `/client.js` 嵌入脚本的形态，并增加了 Mastodon 同步能力。

[English README](README.md)

## 功能

- 将你最新的 Mastodon 状态同步到 Cloudflare D1。
- 存储嘟文文本、发布时间、原文链接和远程媒体 URL。
- 不把图片或视频二进制内容存入 D1。
- 按 Mastodon 状态 ID 去重，避免重复导入。
- 通过 Worker 提供一个公开状态页。
- 提供 `/client.js`，可嵌入 WordPress、Bear Blog 或其他网页。
- 支持 Cloudflare Cron 定时同步，以及受保护的手动同步 API。

## 必需的 Cloudflare Binding

创建一个 D1 数据库，并将它绑定到 Worker：

```text
DB
```

Binding 名称必须是 `DB`。Worker 会在第一次请求时自动创建或迁移所需数据表，不需要单独运行初始化 schema。

## 必需环境变量

```text
ADMIN_PASSWORD=change-me
SESSION_SECRET=long-random-secret
MASTODON_INSTANCE_URL=https://mastodon.social
MASTODON_USERNAME=yourname
ALLOWED_ORIGINS=https://yourname.bearblog.dev,https://your-wordpress.example
```

如果你已经知道 Mastodon 账号 ID，也可以用 `MASTODON_ACCOUNT_ID` 代替 `MASTODON_USERNAME`。

如果需要同步私密、仅关注者可见，或其他需要 token 才能访问的状态，还需要设置：

```text
MASTODON_ACCESS_TOKEN=your-token
```

## 可选环境变量

```text
MASTODON_IMPORT_LIMIT=20
MASTODON_INCLUDE_REPLIES=false
MASTODON_INCLUDE_REBLOGS=false
SITENAME=Status
SITE_DESCRIPTION=A personal Mastodon stream.
API_URL=https://your-worker.example.workers.dev
```

`MASTODON_IMPORT_LIMIT` 每次同步最多支持 40。

## 从 Mac 终端部署

1. 安装或确认 Node.js：

```bash
node --version
npm --version
```

如果缺少这些命令，可以从 [nodejs.org](https://nodejs.org/) 安装，或使用 Homebrew：

```bash
brew install node
```

2. 安装 Wrangler：

```bash
npm install -g wrangler
wrangler --version
```

3. 登录 Cloudflare：

```bash
wrangler login
```

4. 创建 D1 数据库：

```bash
wrangler d1 create mastodon-in-blog
```

复制返回的 `database_id`。配置里的 D1 binding 名称仍然必须保持为 `DB`。

5. 创建本地 Wrangler 配置：

以 `wrangler.toml.example` 为模板：

```bash
cp wrangler.toml.example wrangler.toml
```

编辑 `wrangler.toml`，填写：

- `name`
- `database_id`
- `MASTODON_INSTANCE_URL`
- `MASTODON_USERNAME`
- `ALLOWED_ORIGINS`

Bear Blog 的 `ALLOWED_ORIGINS` 只填写页面 origin，例如 `https://yourname.bearblog.dev` 或你的 Bear Blog 自定义域名。如果也要嵌入 WordPress，把 WordPress origin 用逗号追加进去。

6. 设置 secrets：

```bash
wrangler secret put ADMIN_PASSWORD
wrangler secret put SESSION_SECRET
```

`SESSION_SECRET` 建议使用足够长的随机值。

7. 可选：设置 Mastodon access token。

如果需要 Mastodon token：

```bash
wrangler secret put MASTODON_ACCESS_TOKEN
```

公开账号通常不需要 token。私密、仅关注者可见或需要 token 的内容需要设置它。

8. 部署：

```bash
wrangler deploy
```

部署后打开一次 Worker URL。第一次请求会自动初始化或迁移 D1 数据表。

## 手动同步

登录 `/login` 后，可以手动触发 Mastodon 同步：

```bash
curl -X POST https://your-worker.example.workers.dev/api/sync/mastodon \
  -H "Origin: https://your-worker.example.workers.dev" \
  -H "Cookie: gb_session=..."
```

日常使用时，让 Cron 自动运行即可。

## 嵌入 Bear Blog

创建一个页面，例如 `/status/`，加入：

```html
<div data-gb data-gb-api-url="https://your-worker.example.workers.dev"></div>
<script src="https://your-worker.example.workers.dev/client.js"></script>
```

将你的 Bear Blog origin 加入 `ALLOWED_ORIGINS`，例如 `https://yourname.bearblog.dev` 或 Bear Blog 自定义域名。不要包含 `/status/` 这样的路径。

## 嵌入 WordPress

自托管 WordPress 可以添加一个 Custom HTML block：

```html
<div data-gb data-gb-api-url="https://your-worker.example.workers.dev"></div>
<script src="https://your-worker.example.workers.dev/client.js"></script>
```

部分 WordPress.com 套餐会限制自定义 JavaScript。如果脚本被过滤，请使用自托管 WordPress，或支持自定义脚本的套餐/插件。

同时把 WordPress origin 加入 `ALLOWED_ORIGINS`。

## 嵌入流样式

嵌入脚本会在 `[data-gb]` 容器下插入 markup。你可以在宿主博客里添加 CSS。

常用 class：

| Class | 作用 |
| --- | --- |
| `.gb-widget` | Widget 根元素 |
| `.gb-entries-list` | 列表容器 |
| `.gb-entry` | 单条状态 |
| `.gb-entry-content` | 状态正文 |
| `.gb-entry-meta` | 日期/原文链接行 |
| `.gb-entry-original-link` | Mastodon 原文链接 |
| `.gb-entry-date` | 状态日期 |
| `.gb-load-more-btn` | 加载更多按钮 |

示例：

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

## 数据说明

D1 存储文本、时间戳、Mastodon ID、原文链接和媒体 URL。图片仍由 Mastodon 实例/CDN 托管，除非你另外增加 R2 媒体归档流程。

## License

GNU AGPL v3，跟随上游 MyStatus license。
