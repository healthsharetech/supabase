# HealthShareTech Supabase fork

This is a fork of `supabase/supabase`, used **only** for the self-hosted
Supabase stack under `docker/`. The rest of the monorepo (studio, docs, www)
is upstream and not used by HST. (Upstream monorepo notes are kept below for
the rare case studio code is touched.)

## Branches

| Branch       | Meaning |
| ------------ | ------- |
| `master`     | Pristine mirror of upstream `supabase/supabase` master. No HST commits. |
| `production` | **The deployed branch.** = `master` at last sync + HST customization commits. |
| `rds`        | Abandoned experiment (external RDS, no `db` container). Do not use. |

## Deployment

`bin/deploy` in `reimbursement-tracker-web` deploys by SSHing to the prod
server and running, in `~/supabase/docker` (the `production` branch):
`git pull; op read "<1Password ref>" > .env; docker-compose up -d`.
So `docker/docker-compose.yml` on `production` **is** the live deployment.
The `.env` is managed in 1Password (not `.env.example`).

## Merging upstream

1. `git fetch upstream master` (`https://github.com/supabase/supabase.git`).
2. Work on a branch based on `origin/production`.
3. `git merge upstream/master`. Conflicts are almost always limited to
   `docker/docker-compose.yml` and `.gitignore`. `docker/volumes/api/kong.yml`
   and `docker/volumes/functions/main/index.ts` usually auto-merge — verify
   the HST bits below survived.
4. Take upstream's new structure/image versions; re-apply every HST
   customization below.
5. Validate: `cd docker && docker compose config` must exit 0 and resolve to
   exactly the running-services set below.

## HST customizations (preserve on every upstream merge)

**Disabled services** (commented out in `docker/docker-compose.yml`, not deleted):
- `studio` — gives DB access without a password; never run on prod.
- `api-gw` (Envoy) — upstream's default gateway since 0.8.0. This fork stays
  on Kong. Do not start Envoy next to Kong; both want port 8000. Do not layer
  `docker-compose.kong.yml` either: Kong is already the `kong` service.
- `analytics` (logflare) and `vector` — moved upstream into
  `docker-compose.logs.yml`. Do not enable that override; logs go to
  CloudWatch via `awslogs`.
- `supavisor` — skipped; clients connect directly to Postgres. The commented
  ports are bound to `127.0.0.1`.
- `functions` (edge-runtime) — all edge functions were removed
  (`reimbursement-tracker-web#330`); disabled to reduce attack surface.

**Running services:** `admin, admin-scheduler, admin-worker, auth, db, imgproxy, kong, meta, realtime, rest, storage`.

**Other compose changes:**
- Every running service has an `awslogs` (CloudWatch) `logging:` block
  (`awslogs-region: ${AWS_REGION}`, `awslogs-group: supabase`, `tag`).
- `db` publishes `127.0.0.1:${POSTGRES_PORT}:5432` (supavisor disabled).
  Kong stays on all interfaces so the load balancer can reach it.
- `kong`: image `kong/kong:3.9.3`; `depends_on: studio` commented out;
  `KONG_ROUTER_FLAVOR: expressions`; `KONG_DNS_VALID_TTL` and
  `KONG_DNS_STALE_TTL` both `5` (the stale TTL is the PR #4 fix; upstream
  only set the valid TTL). Network alias `api-gw` so upstream's
  `http://api-gw:8000` URLs still resolve.
- Postgres image is **17** (`supabase/postgres:17.6.1.136`). Do not
  `docker compose up` that image onto the existing PG 15 data directory.
  Run `sudo bash utils/upgrade-pg17.sh` first. See the header of
  `docker/docker-compose.yml`.
- Custom **`admin`** service (`reimbursement-tracker-admin`, Laravel) sits
  where `analytics` was. Built from `docker/admin/{Dockerfile,Caddyfile}`;
  volumes `docker/volumes/admin/{app,caddy_data,caddy_config}`; large
  `ADMIN_*` / Stripe / Plaid / Telnyx / SES / Textract / Mailchimp / Firebase
  env passthrough.
- `auth`: `GOTRUE_EXTERNAL_ANONYMOUS_USERS_ENABLED: false` (hardcoded — an
  empty env value breaks gotrue config parsing); `GOTRUE_MFA_ENABLED: true`;
  debug logging; full `GOTRUE_MAILER_*` subjects/templates/external-hosts/OTP.
- `storage`: S3 backend — `STORAGE_BACKEND: s3`,
  `GLOBAL_S3_BUCKET: ${AWS_S3_BUCKET}`, `REGION: ${AWS_REGION}`,
  `TENANT_ID: storage-single-tenant`. Credentials via EC2 instance IAM role.
- `realtime`: `RLIMIT_NOFILE: ""` — workaround for an EC2 crash.
  `DB_ENC_KEY` is `${REALTIME_DB_ENC_KEY}` with no demo fallback.
  `REGION: us-east-1` is the upstream workaround for missed Postgres Changes
  (realtime#2289), not this host's AWS region.
- `meta` connects as `postgres`, not `supabase_admin` (upstream 0.6.0).

**`docker/volumes/api/kong.yml`:** added `admin` route (`/admin`, `/backend`,
`/livewire`, `/vendor/livewire` → `http://admin:8080`); studio "dashboard"
catch-all commented out; `functions-v1` route commented out (the commented
block tracks upstream's `sb-api-key` translation).

**`docker/volumes/functions/main/index.ts`:** `.env`-file loader patch. Moot
(functions disabled) but kept for minimal diff.

## New upstream env vars to add to the production `.env` (1Password)

Watch for new no-default `${VAR}` refs in running services on each upgrade.
The 2026-05 upgrade introduced:
- `PG_META_CRYPTO_KEY` — **required** by postgres-meta (≥32-char random).
- `IMGPROXY_AUTO_WEBP` — set `true` (replaced `IMGPROXY_ENABLE_WEBP_DETECTION`).
- `S3_PROTOCOL_ACCESS_KEY_ID` / `S3_PROTOCOL_ACCESS_KEY_SECRET` — optional
  (storage S3-protocol API only).

The 2026-10 upgrade (`0eb37adb..564d724e`) adds:
- `REALTIME_DB_ENC_KEY` — **required**. No compose fallback. Existing rows
  were encrypted with the old hardcoded demo value `supabaserealtime`; set
  the 1Password `.env` to the key already in use before deploying.
- `API_EXTERNAL_URL` — existing var. Upstream now expects it to end in
  `/auth/v1` (`GOTRUE_JWT_ISSUER` is that value). Append the suffix once if
  it is missing. Do not double it. Confirm mailer and OAuth callbacks still
  land on `/auth/v1/...`.
- Do not set `COMPOSE_FILE`. The deploy must keep using `docker-compose.yml` alone.

---

# Supabase Monorepo (upstream)

pnpm 10 + Turborepo monorepo. Requires Node >= 22.

## Structure

| Directory         | Purpose                                                      |
| ----------------- | ------------------------------------------------------------ |
| `apps/studio`     | Supabase Studio/Dashboard — Next.js (pages router), React 18 |
| `apps/docs`       | Documentation site                                           |
| `apps/www`        | Marketing website                                            |
| `packages/ui`     | Shared UI components (shadcn/ui based)                       |
| `packages/common` | Shared utilities and telemetry constants                     |
| `e2e/studio`      | Playwright E2E tests for Studio                              |

## Common Commands

```bash
pnpm install                          # install dependencies
pnpm dev:studio                       # run Studio dev server
pnpm test:studio                      # run Studio unit tests (vitest)
pnpm --prefix e2e/studio run e2e       # run Studio E2E tests (playwright)
pnpm build --filter=studio             # build Studio
pnpm lint --filter=studio              # lint Studio
pnpm typecheck                        # typecheck all packages
```

## Conventions

**UI** — import from `'ui'`, use `_Shadcn_` suffixed variants for form primitives. Check `packages/ui/index.tsx` before creating new primitives.

**Styling** — Tailwind only, semantic tokens (`bg-muted`, `text-foreground-light`), no hardcoded colors.

**Language** — Use U.S. English everywhere.

## Studio

Pages router. Co-locate sub-components with parent. Avoid barrel re-export files.

See studio-\* skills for detailed studio conventions.
