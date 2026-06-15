---
project: 10x-cards
researched_at: 2026-06-16
recommended_platform: Cloudflare Workers
runner_up: Netlify
context_type: mvp
tech_stack:
  language: TypeScript (JS family)
  framework: Astro 6 (SSR) + React 19
  runtime: Cloudflare workerd (V8 isolate)
---

## Recommendation

**Deploy on Cloudflare Workers.**

The project is already wired for it — `@astrojs/cloudflare` v13, `output: "server"`, and a `wrangler.jsonc` that points at the Worker server entrypoint — so choosing anything else means swapping the Astro adapter and re-validating the build. Cloudflare passed all five agent-friendly criteria (CLI-first `wrangler`, fully managed isolates, `llms.txt` + markdown docs, deterministic `wrangler deploy`/versioned rollback, and an official MCP server), the developer is already familiar with it (interview Q3), and its free tier (100k requests/day) comfortably covers a low-QPS, small-data MVP. The external data layer — Supabase (auth + DB) and OpenRouter (LLM) — are plain `fetch()` calls from the Worker, so co-location was never needed (interview Q5), and the stateless request/response model (Q1) fits the isolate execution model exactly.

## Platform Comparison

| Platform | CLI-first | Managed/Serverless | Agent-readable docs | Stable deploy API | MCP / Integration | Total |
|---|---|---|---|---|---|---|
| **Cloudflare Workers** | Pass | Pass | Pass | Pass | Pass | **5 Pass** |
| **Netlify** | Pass | Pass | Pass | Pass | Pass | 5 Pass* |
| **Vercel** | Pass | Pass | Pass | Pass | Partial | 4P + 1Part* |
| Fly.io | Pass | Pass | Pass | Pass | Partial | 4P + 1Part* |
| Railway | Pass | Pass | Partial | Pass | Fail | 3P |
| Render | Partial | Pass | Partial | Pass | Fail | 2P |

\* Netlify/Vercel/Fly/Railway all require swapping the Astro adapter away from `@astrojs/cloudflare` and re-testing the build.

**Per-platform notes:**

- **Cloudflare Workers** — `wrangler` covers the full operational loop (`deploy`, `versions list`, `versions deploy <id>@%` for staged rollback, `tail` for logs). Workers are fully managed isolates (no OS/TLS/scaling to manage). Docs publish `llms.txt` + markdown on GitHub. Official `cloudflare/mcp-server-cloudflare` covers docs/Workers/observability. Free tier: 100k req/day; Paid $5/mo → 10M req/mo with higher CPU/subrequest ceilings.
- **Netlify** — Strong on all five (official Netlify MCP, `netlify` CLI). Free Starter tier (100 GB bandwidth, 300 build min/mo) has **no non-commercial restriction**. Cons: requires `@astrojs/netlify`, functions run on Deno with cold starts measured at 0.8–1.5 s after idle.
- **Vercel** — Best-in-class DX and `vercel` CLI, MDX docs on GitHub. Vercel MCP is **beta** (status checked 2026-06). Cons: Hobby tier is **non-commercial only** (a monetized product needs Pro at $20/seat/mo), 10 s function limit, and `@astrojs/vercel` adapter swap.
- **Fly.io** — Container PaaS with `flyctl`, good docs. Cons: **free tier removed** (CC required; realistic small-app cost $8–25/mo), runs the Node adapter as an always-on process — overkill for a stateless app. Multi-region strength is irrelevant here (single region, Q4).
- **Railway** — Clean DX, runs `@astrojs/node` natively with no cold starts, but **no free tier** ($5/mo min, usage-based), weaker agent docs, no first-class MCP.
- **Render** — Free static hosting, but a warm web service is $7/mo (free tier cold-starts/sleeps); deploy is hook/API-driven rather than CLI-centric, no first-class MCP.

### Shortlisted Platforms

#### 1. Cloudflare Workers (Recommended)

Won on zero migration cost (adapter already installed and configured), perfect 5/5 on the agent-friendly criteria, developer familiarity as the tie-breaker, and a free tier that fits the PRD's low-QPS / small-data / medium-user profile. Global edge is a free bonus even though only one region is required.

#### 2. Netlify

The strongest alternative if the team ever wants to leave the edge-isolate runtime behind: equally agent-friendly (official MCP, solid CLI) and its free tier — unlike Vercel's — carries no non-commercial clause. The gap vs. Cloudflare is the `@astrojs/netlify` adapter swap plus Deno-function cold starts, neither of which is justified while the stack is already Cloudflare-native.

#### 3. Vercel

Best developer experience and tightest preview-deploy loop, but loses to both above for this project: the Hobby tier's non-commercial restriction is a poor fit for a product that may monetize, the MCP is still beta, and it also requires an adapter swap. Reach for it only if DX outweighs those caveats.

## Anti-Bias Cross-Check: Cloudflare Workers

### Devil's Advocate — Weaknesses

1. **Workers isolate is not Node.** No `fs`, no `child_process`, limited `node:crypto`. A custom SRS scheduler (FR-009) or any LLM/date dependency that reaches for a Node API can pass `astro dev` yet fail the production build with opaque "Cannot resolve module" errors.
2. **CPU/wall limits (30 s / 128 MB) and subrequest caps** (50/request free, 1000 paid). LLM generation from a long pasted block (FR-003) via OpenRouter can brush against these — made worse because the input-length bound is still an Open Question in the PRD.
3. **The "source text leaves no trace" guarantee collides with Workers Logs / `wrangler tail`** — naive body logging would expose the pasted text to anyone with operator access.
4. **No built-in rate limiting.** The NFR to reject credential-stuffing at scale needs Cloudflare Rate Limiting rules or a Durable Object / KV counter — easy to forget at MVP and not provided by the app code.
5. **Pages-vs-Workers ambiguity.** `tech-stack.md` says `cloudflare-pages`, but the scaffold is a Worker (`main: @astrojs/cloudflare/entrypoints/server`). Deploying via the wrong path (Pages Git integration vs. `wrangler deploy`) produces subtly different env/runtime behavior.

### Pre-Mortem — How This Could Fail

Six months in, the team is stuck. The SRS scheduler (FR-009) was built on a popular npm library that quietly depends on `node:crypto` and an `Intl` polyfill; it worked partially in `astro dev` (workerd) but broke in production under load with unreadable build errors that cost days. OpenRouter generations for long texts occasionally exceeded the Worker CPU/wall limit, returning truncated candidate sets; without a server-side input-length bound (still an Open Question), some requests silently failed, dragging down the "75% acceptance" metric because users saw broken generations. Nobody implemented credential-stuffing protection, so an early scripted attack inflated Supabase auth usage. And because they leaned on Pages Git auto-deploy instead of `wrangler deploy`, env-var binding differed between preview and prod, leaking a misconfigured `SUPABASE_KEY` into a preview build. None of these are Cloudflare's fault — they're the edge runtime's sharp edges meeting an under-specified PRD.

### Unknown Unknowns

- **`npm run dev` is not `wrangler dev`.** Astro 6 runs `astro dev` on the workerd runtime (Cloudflare Vite plugin), so fidelity is high — but bindings (KV, Rate Limiting, secrets from `.dev.vars`) need explicit local config, and a dependency importing a Node API can pass dev yet fail the production build.
- **Secrets live in two places.** Locally they're in `.dev.vars`; in production they must be set as Worker **secrets** (`wrangler secret put`), not just env files. CI needs them too — and the CI workflow currently triggers on `master` while the repo default branch is `main`, so the pipeline meant to validate the deploy does not actually run.
- **Free plan is 100k requests/day with a daily reset**, plus 50 subrequests/request. The "proxy to LLM" pattern (one generation = several subrequests to OpenRouter) can hit the subrequest cap faster than the request count suggests. Paid ($5/mo) lifts this to 10M/mo + 1000 subrequests.
- **The "no trace" guarantee interacts with observability.** Enabling Workers Logs with request-body logging, or logging the payload while debugging, turns the pasted source text operator-visible — a direct PRD guardrail violation.

## Operational Story

- **Preview deploys**: `wrangler versions upload` creates a preview (versioned) deployment with its own `*.workers.dev` preview URL without promoting it to production; promote with `wrangler versions deploy`. Preview URLs are public by default — gate sensitive previews with Cloudflare Access if needed. (Avoid the Pages Git-integration preview path; this project is a Worker.)
- **Secrets**: production secrets via `wrangler secret put SUPABASE_URL` / `SUPABASE_KEY` / `OPENROUTER_API_KEY` (stored encrypted in Cloudflare, readable only by account members with access); local dev via `.dev.vars` (gitignored); CI build reads them from GitHub repository secrets. Rotate by re-running `wrangler secret put` and redeploying.
- **Rollback**: `wrangler versions list` to find the prior version id, then `wrangler versions deploy <previous-id>@100%` (or staged `@10%`/`@90%`). Time-to-revert is seconds — it re-points traffic, no rebuild. Caveat: Supabase schema migrations do **not** roll back with the Worker; reverting code does not revert DB state.
- **Approval**: a human approves promoting a version to 100% production traffic, rotating the primary `SUPABASE_KEY`, and any Supabase migration that drops/alters columns. An agent may unattended: build, `wrangler versions upload` (preview), tail logs, and run a staged canary deploy.
- **Logs**: `wrangler tail --status error` (live, read-only) and `wrangler tail --search "<term>"` for filtering; `wrangler deployments view` / `wrangler versions list` for deploy history. Workers Logs (enabled in `wrangler.jsonc` `observability`) gives persisted structured logs — log JSON, never the pasted source text.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Node-only dependency (SRS lib, crypto, date) breaks the production build | Devil's advocate / Pre-mortem | M | H | Vet every new dep for Node-API usage; prefer Web-standard libs; test with `wrangler deploy` to a preview before merging, not just `astro dev`. |
| LLM generation exceeds Worker CPU/wall/subrequest limits on long input | Devil's advocate / Pre-mortem | M | H | Resolve PRD Open Question #1 (max input length) and enforce a server-side bound before calling OpenRouter; stream the response (ReadableStream) for the "no freeze" NFR. |
| Pasted source text leaked into Workers Logs / `wrangler tail` | Devil's advocate / Unknown unknowns | M | H | Never log request bodies; assert structured logging excludes the `text` field; review observability config before enabling body logging. |
| No credential-stuffing / brute-force protection on auth | Research finding (PRD NFR) | M | M | Add Cloudflare Rate Limiting rule on `/api/auth/*`, or a KV/Durable-Object counter; allow a few retries per IP+account, reject scaled attempts. |
| Deploy via Pages dashboard instead of `wrangler deploy` causes env/runtime drift | Devil's advocate / Unknown unknowns | M | M | Treat the project as a Worker; deploy only with `wrangler deploy`; correct the stale `cloudflare-pages` hint in `tech-stack.md`. |
| CI does not run (workflow on `master`, default branch is `main`) | Unknown unknowns | H | M | Fix `.github/workflows/ci.yml` to target `main` (or rename the branch) so lint+build gate the deploy. |
| Free-tier daily request/subrequest cap hit during a usage spike | Research finding | L | M | Monitor usage; upgrade to Workers Paid ($5/mo) before launch traffic; the LLM-proxy subrequest count is the limit to watch, not raw requests. |
| Supabase migration can't be rolled back with a code revert | Operational (rollback caveat) | L | M | Make migrations additive/backward-compatible; never couple a destructive migration to a deploy that might be reverted. |

## Getting Started

These commands target the versions pinned in this repo (`@astrojs/cloudflare` v13, `wrangler` v4, Astro 6 — `astro dev` already runs on workerd, so no separate `wrangler pages dev` step is needed):

1. **Authenticate wrangler**: `npx wrangler login` (opens a browser once to authorize the Cloudflare account).
2. **Set production secrets**: `npx wrangler secret put SUPABASE_URL`, then the same for `SUPABASE_KEY` and `OPENROUTER_API_KEY`. (Local dev keeps these in `.dev.vars`.)
3. **Build**: `npm run build` (produces the Worker + `dist/` static assets served per `wrangler.jsonc`).
4. **Deploy a preview first**: `npx wrangler versions upload` → open the preview URL → verify auth and a generation round-trip against real Supabase/OpenRouter.
5. **Promote to production**: `npx wrangler versions deploy <version-id>@100%` (or `npx wrangler deploy` for a direct production deploy). Roll back with `npx wrangler versions deploy <previous-id>@100%`.

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration
- CI/CD pipeline setup
- Production-scale architecture (multi-region, HA, DR)
