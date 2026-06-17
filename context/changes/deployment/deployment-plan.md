# Cloudflare Workers — Integration & Deployment Plan (10x-cards)

## Context

`context/foundation/infrastructure.md` selected **Cloudflare Workers** as the MVP platform and the
repo is already wired for it (`@astrojs/cloudflare` v13.5, `output: "server"`, a `wrangler.jsonc`
pointing at the Worker server entrypoint, `nodejs_compat` enabled). This plan turns that decision
into a first live deployment, end to end, while closing the gaps the infra doc's risk register
flagged. The goal: a verified production deploy on `*.workers.dev` with auth + a runtime round-trip
against the hosted Supabase project, a working rollback path, and Cloudflare-native auto-deploy.

**Scope decisions (confirmed with user):**
- **OpenRouter — deferred.** Not implemented in code; no env var or secret wired now. Add when the LLM generation feature lands.
- **Worker rename → `10x-cards`** (currently the generic `10x-astro-starter`).
- **Auto-deploy via Cloudflare Workers Builds (Git-connected), not GitHub Actions.** Cloudflare builds + deploys on push to the production branch and uploads a preview version for non-production branches. **The existing GHA workflow (`.github/workflows/ci.yml`) is removed entirely** — no GHA deploy, no Cloudflare token in GitHub. (Trade-off: Workers Builds does **not** run ESLint, so lint no longer gates merges in CI; the husky/lint-staged pre-commit hook still runs `eslint --fix` on staged `*.{ts,tsx,astro}` locally.)
- **Hardening (rate limiting, Supabase migration + RLS) — documented as non-blocking follow-up steps**, not gating the first deploy.

**Deployment is a Worker, NOT Pages.** Deploy only via `wrangler deploy` / `wrangler versions` / Workers Builds. Do not use the Pages Git-integration path — it produces env/runtime drift (infra doc Risk Register row 5).

> This document is the deploy artifact / audit trail (the 10xDevs hand-off, relocated here at `context/changes/deployment/deployment-plan.md`). Keep checkboxes updated as phases complete.

---

## Verified current state

| Item | State |
|---|---|
| `wrangler.jsonc` | name `10x-astro-starter`, `main: @astrojs/cloudflare/entrypoints/server`, `compatibility_date: 2026-05-08`, `compatibility_flags: ["nodejs_compat"]`, `assets`→`./dist`, `observability.enabled: true`. No vars/secrets/bindings. |
| `astro.config.mjs` `env.schema` | only `SUPABASE_URL`, `SUPABASE_KEY` (server / secret / `optional: true`). |
| Secrets handling | `src/lib/supabase.ts` reads them via `astro:env/server`, returns `null` if unset (null-safe everywhere). |
| `.dev.vars` / `.dev.vars.example` | **Neither exists.** `.dev.vars` is gitignored (`.gitignore:26`). |
| CI (`.github/workflows/ci.yml`) | triggers on **`master`** (repo default is `main` → CI doesn't run); steps: `npm ci` → `astro sync` → lint → build. No deploy step. **→ to be removed; deploy moves to Workers Builds.** |
| Pre-commit | husky + lint-staged runs `eslint --fix` on staged `*.{ts,tsx,astro}` (local lint gate survives GHA removal). |
| Workerd compatibility | Clean — no Node-only APIs in `src/`. |
| OpenRouter | Not implemented. |
| Rate limiting / Supabase migrations | None. |
| Versions | Astro 6.3.1, `@astrojs/cloudflare` 13.5.0, wrangler 4.90.0, `@supabase/ssr` 0.10.3, Node 22.14.0. |

---

## Prerequisites — CLI & Supabase configuration

One-time setup required **before Phase 0**. Both CLIs are already project devDependencies (`wrangler` ^4.90, `supabase` ^2.23), so use `npx` — no global installs.

### A. Tooling baseline

- [x] Node `22.14.0` (`.nvmrc`) — `nvm use` (or install that version); npm ships with it.
- [x] `npm install` once so the `wrangler` and `supabase` binaries resolve via `npx`.
- [ ] Docker Desktop running — required **only** if you run the local Supabase stack (`supabase start`); not needed if local dev points at the hosted project.

### B. Cloudflare / wrangler CLI

- [x] `npx wrangler login` → browser OAuth; authorizes the CLI against your Cloudflare account. **Done** — logged in via OAuth token (sstachurski@gmail.com).
- [x] `npx wrangler whoami` → confirm email + account. **Done** — account `sebiusz` (`edfe340ba20304bf394d478e9dc3f218`). Single account, no pinning needed.
- [ ] Dashboard → Workers & Pages → Account → **Subdomain**: ensure a `*.workers.dev` subdomain is registered — the first deploy fails without one.
- Note: deploy auth for **Workers Builds** is handled by the Cloudflare GitHub app (Phase 6), so no `CLOUDFLARE_API_TOKEN` is needed here. A token is only required if you later script non-interactive `wrangler deploy` outside Builds.

### C. Supabase — hosted project (production)

- [x] Create (or identify) the hosted project at supabase.com — it supplies the values used as Worker secrets.
- [ ] Settings → API → copy **Project URL** → `SUPABASE_URL`, and the **anon / publishable** key → `SUPABASE_KEY`. Do **not** use the service-role/secret key (the cookie-SSR pattern needs the public anon key).
- [ ] Note the **project ref** (the subdomain in the URL, e.g. `abcdefgh`) for linking the CLI below.
- These values feed Phase 2 (`wrangler secret put`) and the hosted Auth-URL alignment in Phase 3.

### D. Supabase CLI — link & local stack (recommended)

- [ ] `npx supabase login` → access token via browser.
- [ ] `npx supabase link --project-ref <ref>` → links this repo's `supabase/config.toml` (project_id `10x-astro-starter`, Postgres major v17) to the hosted project; required before pushing migrations in Phase 8.
- [ ] **Local stack (needs Docker):** `npx supabase start` boots Postgres + Auth + Studio locally (API on `127.0.0.1:54321`, DB on `54322`) and prints a local **API URL** + **anon key**. `npx supabase stop` shuts it down.
  - **Pick ONE source for `.dev.vars` (Phase 1):** the **local** stack values (offline, fast — safer dev default) or the **hosted** project values (exercises the real prod data layer). Whichever you choose for dev, verify against the hosted project at least once via the Phase 4 preview.
  - **Edge-case support — Postgres version match:** `config.toml` pins `major_version = 17`; if the hosted project runs a different major version, `supabase db` commands warn/fail. Align them before linking.

---

## Phase 0 — Pre-flight (read-only)

- [ ] `npx wrangler --version` → confirm v4.x.
- [ ] `npx wrangler whoami` → confirm authenticated account + that a `*.workers.dev` subdomain is registered for the account.
  - **Edge-case support:** if `whoami` shows no account, run `npx wrangler login` (browser auth). If the account has **no `workers.dev` subdomain yet**, the first `wrangler deploy` will fail asking you to register one in the dashboard (Workers & Pages → Account subdomain). Register it before deploying.
- [ ] `npm run build` locally once to confirm a clean baseline build before any config change.

## Phase 1 — Local config changes

Files: `wrangler.jsonc`, new `.dev.vars`, new `.dev.vars.example`.

- [ ] **Rename the worker:** in `wrangler.jsonc` set `"name": "10x-cards"`. This becomes the `10x-cards.<subdomain>.workers.dev` URL. Doing this before the first deploy avoids an orphaned `10x-astro-starter` worker.
- [ ] Create `.dev.vars` (gitignored) with real local values:
  ```
  SUPABASE_URL=...
  SUPABASE_KEY=...
  ```
- [ ] Create `.dev.vars.example` (committed, no real values) mirroring the keys — so the local-secrets contract is discoverable. Keep it in sync with `.env.example`.
- [ ] `npm run dev` (runs on workerd) → confirm auth flow works locally reading from `.dev.vars`.
  - **Edge-case support — secrets live in two places:** `.dev.vars` is local-only; production needs them set as Worker **secrets** (Phase 2). `.env`/`.env.example` are for Node tooling/CI build. Do not assume setting one covers the others.

## Phase 2 — Production secrets (human-controlled)

- [ ] `npx wrangler secret put SUPABASE_URL`
- [ ] `npx wrangler secret put SUPABASE_KEY`
  - Stored encrypted on Cloudflare; persist across deploys (don't need re-setting per deploy).
  - **Rotation is a human action** (infra doc approval policy): re-run `wrangler secret put` + redeploy.
  - **OpenRouter:** deferred — do not add `OPENROUTER_API_KEY` now.

## Phase 3 — Supabase hosted-project alignment (external integration)

The local `supabase/config.toml` (`site_url: http://127.0.0.1:3000`, email confirmations disabled) governs **local** Supabase only. The **hosted** Supabase project that production talks to has independent Auth settings that must be aligned, or auth redirects/email links will break.

- [ ] In the hosted Supabase dashboard → Authentication → URL Configuration: set **Site URL** and add **Redirect URLs** for the production Worker domain (`https://10x-cards.<subdomain>.workers.dev`), plus any custom domain later.
- [ ] Confirm the `SUPABASE_KEY` used as a Worker secret is the **anon/publishable** key (cookie-SSR pattern), not the service-role key.
- [ ] **Edge-case support — email confirmation:** local config disables email confirmation; the hosted project may require it. If signup expects a confirmation link, ensure the redirect target resolves on the Worker domain (see `src/pages/auth/confirm-email.astro`, which only auto-confirms under `import.meta.env.DEV`).

## Phase 4 — Build & preview deploy (agent may run unattended)

- [ ] `npx astro sync` then `npm run build` (Worker + `dist/` assets).
- [ ] `npx wrangler versions upload` → creates a **preview** versioned deploy with its own `*.workers.dev` preview URL (does NOT promote to production).
- [ ] Smoke-test the preview URL: load a page, sign in, exercise a protected route (`/dashboard`), verify a real Supabase round-trip.
  - **Edge-case support — Node-only dep at build time:** if `npm run build` fails with "Cannot resolve module" / Node-API errors after a future dependency change, the culprit is a workerd-incompatible dep that passed `astro dev` but fails the production build (infra doc Risk #1). Always validate new deps through `wrangler versions upload`, not just `astro dev`.
  - **Edge-case support — preview URLs are public.** If a preview must stay private, gate it with Cloudflare Access.

## Phase 5 — Promote to production

- [ ] `npx wrangler versions deploy <version-id>@100%` (staged canary `@10%`→`@100%` optional).
  - Alternatively `npx wrangler deploy` for a direct production deploy.
- [ ] Verify production URL: auth + protected route + Supabase round-trip.
- [ ] **Rollback drill (do once now):** `npx wrangler versions list` → `npx wrangler versions deploy <previous-id>@100%`. Re-points traffic in seconds, no rebuild.
  - **Caveat:** a code rollback does **not** revert Supabase schema. Keep migrations additive/backward-compatible (infra doc Risk #8).

## Phase 6 — Auto-deploy via Cloudflare Workers Builds (Git-connected)

Deploy automation is **native Cloudflare**, not GitHub Actions. Workers Builds connects the repo, builds on Cloudflare's runners, deploys the production branch, and uploads a preview version for every other branch.

- [ ] **Remove the GitHub Actions workflow:** delete `.github/workflows/ci.yml` (it triggers on the wrong branch and is superseded). Lint still runs locally via the husky/lint-staged pre-commit hook.
- [ ] **Install the Cloudflare GitHub app** on this repository (prerequisite for the Git connection).
- [ ] In the Cloudflare dashboard → the `10x-cards` Worker → **Settings → Builds → Connect**: select this GitHub repo.
  - **Critical:** the dashboard Worker name **must equal** `name` in `wrangler.jsonc` (`10x-cards`, set in Phase 1) or the build fails. This is why the rename happens before connecting.
- [ ] Configure build settings:
  - **Build command:** `npm run build`
  - **Deploy command (production branch):** `npx wrangler deploy` (default)
  - **Non-production branch deploy command:** `npx wrangler versions upload` (default) → automatic preview URL per branch, no production promotion (preserves the human-on-promotion posture from infra doc).
  - **Production branch:** set to `main`.
  - **Root directory:** repo root (default).
- [ ] **Variables & secrets — two distinct buckets (don't conflate):**
  - *Runtime* secrets (`SUPABASE_URL`, `SUPABASE_KEY`) are Worker secrets from Phase 2 (Settings → Variables & Secrets). They persist across Workers Builds deploys and are **not** re-pushed per build — rotation stays human-only.
  - *Build* variables (Settings → Builds → Build variables) are build-time only and **not** accessible at runtime. The build does **not** need the Supabase secrets (they're `optional: true` in `env.schema`), so leave build variables empty unless a future build step requires one.
  - **Edge-case support:** if `npm run build` ever starts validating a required secret, add it as a **build variable**, not expecting the runtime secret to be visible at build time.

## Phase 7 — Observability & verification

- [ ] `npx wrangler tail --status error` to watch live errors during the first production traffic.
- [ ] Confirm `observability.enabled: true` is producing Workers Logs.
  - **Edge-case support — "no trace" guardrail (infra doc Risk #3):** never log request bodies. When OpenRouter/generation is later added, assert structured logging excludes the pasted `text` field before enabling any body logging.

## Phase 8 — Follow-up hardening (non-blocking, tracked, not part of first deploy)

- [ ] **Auth rate limiting** on `/api/auth/*` — Cloudflare Rate Limiting rule or a KV/Durable-Object counter (no app-level protection exists today; infra doc Risk #4).
- [ ] **Baseline Supabase migration + RLS:** create `supabase/migrations/<YYYYMMDDHHmmss>_init.sql` (dir doesn't exist yet) and enable RLS on every new table, per project Tripwires.
- [ ] **Free-tier ceilings:** monitor request/subrequest usage; upgrade to Workers Paid ($5/mo) before launch traffic — watch the LLM-proxy subrequest count, not raw requests (infra doc Risk #6). Relevant once OpenRouter lands.
- [ ] **OpenRouter wiring** (when the generation feature is built): declare `OPENROUTER_API_KEY` in `astro.config.mjs` `env.schema`, add to `.dev.vars(.example)`, and `wrangler secret put OPENROUTER_API_KEY`.

---

## Files to modify

| File | Change |
|---|---|
| `wrangler.jsonc` | `name` → `10x-cards`. |
| `.dev.vars` (new, gitignored) | real local Supabase values. |
| `.dev.vars.example` (new, committed) | key template, no values. |
| `.github/workflows/ci.yml` | **delete** — deploy moves to Cloudflare Workers Builds; no GHA. |

No new GitHub secrets and no Cloudflare API token in GitHub are needed — the Git connection is owned by the Cloudflare GitHub app.

No source/runtime code changes are needed for the first deploy — the app is already workerd-clean and null-safe.

## Verification (end-to-end)

1. **Local:** `npm run dev` → sign in, hit `/dashboard` → succeeds reading `.dev.vars`.
2. **Build:** `npm run build` clean.
3. **Preview:** `wrangler versions upload` → preview URL: auth + `/dashboard` + Supabase round-trip OK.
4. **Production:** `wrangler versions deploy <id>@100%` → same checks on prod URL.
5. **Rollback:** `wrangler versions list` → deploy previous id → traffic re-points in seconds.
6. **Workers Builds:** push a non-`main` branch → Cloudflare builds + uploads a preview version with its own preview URL; merge/push to `main` → Cloudflare runs `wrangler deploy` to production. Check the build log in the dashboard (Worker → Builds).
7. **Logs:** `wrangler tail --status error` shows no errors during the smoke test.
