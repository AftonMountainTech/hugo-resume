# hugo-resume

<!-- BEGIN cloudflare-guardrails -->
## Cloudflare guardrails

These rules apply to any Cloudflare work in this repo. If a task seems to need
breaking one of them, stop and ask Ben.

### Hard rules
- **Never deploy.** Don't run `wrangler deploy`, `wrangler versions upload`,
  `wrangler versions deploy`, `wrangler rollback`, `wrangler triggers deploy`,
  `wrangler delete`, or `cf deploy`/`cf dev`/`cf build`. Production deploys
  happen only through Workers Builds when Ben merges to the production branch.
- **Never create, delete or modify Cloudflare resources** (KV, D1, R2, Queues,
  Durable Object namespaces, DNS, routes, custom domains, tokens, account
  settings) from the CLI, the API or MCP. Propose the change in the PR instead.
- **Never use or look for Cloudflare credentials.** That covers the Global API
  Key, `wrangler login` OAuth, `CLOUDFLARE_API_TOKEN`, `~/amt/secrets`, and
  `.dev.vars`. If you hit an auth error, report it and don't work around it.
- **Never change the plan or billing.** Upgrading to Workers Paid needs Ben's
  explicit approval, because Paid has no spend cap.
- **Read-only is fine:** `wrangler dev` (local), `wrangler types`,
  `wrangler deploy --dry-run`, `wrangler check`, docs lookups.

### Infrastructure changes need Ben's review
Any change to these files counts as an **infrastructure change**. Call it out
at the top of the PR description, with what it does and why:
- `wrangler.jsonc` / `wrangler.toml`: bindings, routes, custom domains,
  `limits`, `workers_dev`, `preview_urls`, `compatibility_date`/flags, crons,
  DO `migrations`/exports, `env.*` blocks
- `package.json` scripts that build or deploy
- `.github/`, build config, git hooks
- D1 migration files

### Config rules (wrangler.jsonc)
- Set `"workers_dev": false` and `"preview_urls": false` explicitly. Both
  default to **public** and bypass the custom domain's WAF and Access rules.
  Toggling them in the dashboard gets undone by the next deploy.
- Every binding has an explicit ID or name. A binding with no ID makes
  `wrangler deploy` **auto-create** a billable resource.
- `vars`, secrets and bindings are **not inherited** by `env.*` blocks.
  Redeclare them in each env. A DO binding in another env points at different
  storage. Don't add envs unless asked.
- Declare `secrets.required`, so a deploy fails loudly when a secret is
  missing. Secrets go in via `wrangler secret put` (Ben runs it), never `vars`
  or source.
- Keep `compatibility_date` current. Generate `Env` with `wrangler types`, and
  never hand-write it.
- Prefer a custom domain when the Worker is the whole site. A route needs an
  existing proxied DNS record. A custom domain creates its own DNS record, so
  a leftover CNAME conflicts with it. Renaming a Worker leaves the domain
  bound to the old name.
- Set `observability` with a `head_sampling_rate` below 1 for busy Workers.
  Logs and traces are billed per GB from 2026-12-01. Don't log request bodies.
- Ship Durable Object class changes (new/renamed/deleted classes, migrations)
  in **their own PR**. They can't be rolled back. A `deleted_classes`
  migration destroys all data for that class.

### Code rules: loops are what cause five-figure bills
Every large 2026 Cloudflare bill we found was a loop in app code, several of
them written by AI agents: DO alarms re-arming, queue consumers re-enqueueing,
cron or retry storms, D1 full scans.
- **DO alarms:** check `getAlarm()` before `setAlarm()`. Never schedule
  `setAlarm(Date.now())`, and use a minimum delay. Make sure there's a
  terminal condition. Clean up with `deleteAll()` (on compat dates before
  2026-02-24, call `deleteAlarm()` too).
- **Queues / crons / retries:** bounded retries with backoff, idempotent jobs,
  and a consumer must never enqueue to its own queue without a depth or
  attempt cap.
- **Self-fetch:** a Worker must never `fetch()` its own hostname. Use Service
  Bindings, and carry a depth guard if recursion is possible.
- **D1 bills by rows *scanned*.** Index every column used in `WHERE`, `JOIN`
  or `ORDER BY`. Check new queries with `EXPLAIN QUERY PLAN`, and look at
  `meta.rows_read`. No unbounded `SELECT`s or `COUNT`s on hot paths.
  Paginate.
- **KV:** it's eventually consistent (60s+) and allows 1 write per second per
  key. Never use it for counters, locks or anything read-after-write.
- **Crawlers:** serve static files as assets, which is free and skips the
  Worker. Cache read-heavy responses. Make sure filter, sort and pagination
  URLs can't explode into uncached Worker or D1 work.
- **Workers basics:** no floating promises (`await`, `return`, or
  `ctx.waitUntil`), no module-level mutable state (isolates are shared across
  requests), stream large bodies (128 MB memory cap), Web Crypto for tokens,
  never `Math.random()`.
- On Free, limits fail as errors (requests 1027, CPU 1102, D1 daily caps since
  2026-09-01). Handle them gracefully rather than retrying harder.

### Data safety
- Never delete D1 databases, R2 buckets or objects, KV namespaces, or DO
  data. Deleted D1 can be undeleted for only 7 days.
- D1 Time Travel restores **overwrite in place**. Retention is 7 days on Free
  and 30 on Paid. Before a risky migration, say so in the PR and ask Ben for
  an export.
- Apply D1 migrations only by Ben's hand (`--remote`). Local SQLite can
  behave differently from production.
- A code rollback does **not** roll back data or schema. Every schema change
  must be backward-compatible with the previous code version.
- No public R2 buckets without a reviewed reason. R2 has no free-tier hard
  stop.

### Secrets hygiene
- Gitignore `.dev.vars*`, `.env*` and `.wrangler/`. Local Secrets Store values
  sit **unencrypted** in `.wrangler/state`.
- Never print, log or commit secret values. If you find one committed, stop
  and tell Ben.

### Other traps
- Preview builds can run unreviewed branch code with a broad build token.
  Legacy previews and a `wrangler versions upload` preview command both use
  **production** bindings. Keep preview builds off unless Ben enables Worker
  Previews with separate preview bindings.
- Email Routing: plus-addressed mail (`user+tag@`) reportedly skips Email
  Workers silently. `forward()` works only to verified destinations. Inbound
  mail must pass SPF or DKIM and DMARC. Wrap handlers in try/catch, because a
  CPU-exceeded handler may lose mail.
- Bot Fight Mode returns 403 to webhooks (Stripe, monitors), and Free can't
  exempt them. Don't enable it on zones that receive webhooks.
- Pages → Workers migrations: `pages_build_output_dir` → `assets`, add
  `compatibility_date`, disconnect the Pages git integration, repoint DNS.
  Treat it as an infrastructure change.
<!-- END cloudflare-guardrails -->
