# Notes for Claude — joeblew999/d1-manager

This is **joeblew999's fork** of [neverinfamous/d1-manager](https://github.com/neverinfamous/d1-manager).

## Why this fork exists

d1-manager is a self-hostable Cloudflare Worker that admins **every D1 database
on the operator's CF account** (its API token is account-scoped, not DB-
scoped). It's account-level ops infra, not project-level — used by the
joeblew999 ecosystem (`authz-core`, `auth-service`, `remy-sport`, etc.) for a
shared GUI over D1 with GitHub SSO via Cloudflare Access.

We forked because:

1. **The deploy is stateful.** Upstream's `wrangler.toml` references their
   custom domain `d1.adamic.tech` and their D1 `database_id`. Patching those at
   deploy time was the original plan — it's brittle across upstream releases.
   Forking + templating once is cleaner.
2. **We layer the joeblew999 conventions** (mise tasks, `config/<env>.env`,
   fnox-managed secrets, `cf:provision`-style automation, `wrangler.toml`
   generated from `wrangler.toml.template` via `envsubst`) — same shape as
   `joeblew999/auth-service`. Operator experience is consistent across all
   our Workers.
3. **No engine changes.** All upstream `worker/`, `web/`, build pipeline,
   features (R2 backups, BackupDO, AI search, hourly cron, observability)
   are kept exactly as upstream ships them. Cherry-pick upstream bumps via
   `git merge upstream/main` when desired.

## Operator runbook (one-time)

```sh
# 1. Configure
cp config/template.env config/production.env
$EDITOR config/production.env       # set WORKER_NAME + CF_SUBDOMAIN

# 2. Provision CF resources (R2 bucket + metadata D1 + apply schema.sql).
#    Idempotent — writes new IDs back into config/production.env.
mise run cf:provision

# 3. Set up Cloudflare Access (one-time, in dashboard)
#    - Zero Trust → Settings → Authentication → add GitHub OAuth as IdP.
#    - Zero Trust → Access → Applications → create Application gating:
#      <WORKER_NAME>.<CF_SUBDOMAIN>.workers.dev
#    - Copy POLICY_AUD (Application Audience tag).
#    - Copy TEAM_DOMAIN (https://<team>.cloudflareaccess.com — note: https://).

# 4. Generate a CF API token: Account → D1 → Edit. Copy.

# 5. Stash secrets in fnox (this also makes them available to other forks).
fnox set ACCOUNT_ID         # paste your account id
fnox set API_KEY            # paste the API token from step 4
fnox set TEAM_DOMAIN        # paste with https://
fnox set POLICY_AUD         # paste the audience tag

# 6. Push fnox secrets to the Worker's secret store.
mise run secrets:put-cf

# 7. Deploy.
mise run 10-deploy

# 8. Verify Cloudflare Access is gating it.
mise run prove:deployed     # should show HTTP/2 302 to a CF Access challenge URL
```

After this you visit `https://<WORKER_NAME>.<CF_SUBDOMAIN>.workers.dev`,
GitHub-OAuth through CF Access, see the d1-manager UI listing every D1 on
your account.

## File layout (added by this fork)

| Path | Status | Purpose |
|---|---|---|
| `mise.toml` | added | Task runner — phased deploy lifecycle, fnox secret sync, env-driven wrangler.toml gen. |
| `config/template.env` | added | Schema doc for `config/<env>.env`. Gitignored real env files derive from it. |
| `wrangler.toml.template` | added | Template that mise/envsubst expands into `wrangler.toml`. |
| `wrangler.toml` | gitignored | Generated. Do not edit. |
| `config/*.env` | gitignored | Per-env values. Real secrets come from fnox, not these files. |
| `.gitignore` | edited | Added `config/*.env` (excl template) + `wrangler.toml`. |
| `CLAUDE.md` | added | This file. |

Everything else under `worker/`, `web/`, `package.json`, etc. is **untouched** from upstream — to make rebases trivial.

## Decisions you should not reopen

### Why a fork instead of patching at deploy time

The upstream `wrangler.toml` couples to specific resources (R2 bucket name,
D1 id, custom domain). Trying to `sed` those during a downstream task means
tracking changes between upstream releases. Forking once + templating
swaps that for a one-time `git merge upstream/main` when you want the
features they ship.

### Why fnox + GitHub instead of just `wrangler secret put` directly

Three reasons:
1. Centralised. The same `ACCOUNT_ID`, `API_KEY` are needed by `auth-service`,
   `authz-core`, `remy-sport` — fnox holds them once. Don't duplicate.
2. Avoids wrangler's 24h OAuth-token expiry (the API_KEY in fnox is forever).
3. `secrets:put-cf` becomes an idempotent task you can re-run without re-entering values.

### Why account-scoped API token, gated by CF Access

d1-manager admins **every** D1 on the account. The blast radius of getting
through this URL is account-wide. CF Access (GitHub OAuth) is the gate —
without it, anyone with the URL gets account-wide D1 admin. Treat
`POLICY_AUD` setup as load-bearing.

## Upgrading from upstream

```sh
git remote add upstream https://github.com/neverinfamous/d1-manager.git
git fetch upstream
git merge upstream/main             # resolve conflicts (probably just CHANGELOG.md and lock files)
mise run wrangler:gen               # re-render wrangler.toml from template
mise run 10b-redeploy
```

Upstream releases monthly-ish. Bumps that touch `wrangler.toml` (new bindings,
removed routes, etc.) need to be reflected in `wrangler.toml.template` —
re-diff after merge.

## What this fork is not for

- Authz logic — that's `joeblew999/authz-core`. d1-manager only knows D1 rows.
- Auth — that's `joeblew999/auth-service`. d1-manager uses CF Access to gate, doesn't run sessions itself.
- The remy-sport biz model — that's `joeblew999/remy-sport-biz`. d1-manager queries any DB but doesn't understand schemas.
