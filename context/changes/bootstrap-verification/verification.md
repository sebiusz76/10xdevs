---
bootstrapped_at: 2026-06-15T00:00:00Z
starter_id: 10x-astro-starter
starter_name: "10x Astro Starter (Astro + Supabase + Cloudflare)"
project_name: 10x-cards
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: "npm audit --json"
---

## Hand-off

Verbatim copy of `context/foundation/tech-stack.md` frontmatter:

```yaml
starter_id: 10x-astro-starter
package_manager: npm
project_name: 10x-cards
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: false
```

### Why this stack

A solo learner shipping a 3-week after-hours MVP for AI-generated flashcards
needs a battle-tested, agent-friendly starter that handles auth, database, and
edge deploy out of the box so the limited build time goes into the product's
real rule (text → reviewable Q/A candidates), not plumbing. The 10x Astro
Starter (Astro + React + TypeScript + Tailwind + Supabase + Cloudflare) is the
recommended default for `(web, js)` and clears all four agent-friendly gates;
Supabase covers email+password auth (FR-001/002) and flashcard storage, while
project-wide TypeScript and Zod boundaries give a clean surface for the LLM
generation call (FR-003/004). Auth and AI feature flags are set; payments,
realtime, and background jobs are out of scope per the PRD non-goals. The LLM
integration and the "source text leaves no trace" guarantee are not bundled by
the starter and remain custom code. Bootstrapper confidence is first-class —
mostly-smooth scaffolding with occasional manual steps. CI runs on GitHub
Actions with auto-deploy-on-merge, matching the starter's default shape.

## Pre-scaffold verification

| Signal      | Value                                                      | Severity | Notes                                            |
| ----------- | ---------------------------------------------------------- | -------- | ------------------------------------------------ |
| npm package | not run                                                    | n/a      | cmd_template starts with `git clone` — npm step skipped per spec |
| GitHub repo | przeprogramowani/10x-astro-starter last pushed 2026-05-17  | fresh    | from card.docs_url; ~1 month before run          |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`
**Strategy**: git-clone
**Exit code**: 0
**Files moved**: 19
**Conflicts (.scaffold siblings)**: CLAUDE.md.scaffold, .vscode.scaffold
**.gitignore handling**: append-merged (cwd lines kept; starter lines appended under `# from 10x-astro-starter`)
**.bootstrap-scaffold cleanup**: deleted (cloned `.git/` removed before move-up so upstream history did not leak)

Files moved silently (no cwd conflict): `.env.example`, `.github/`, `.husky/`, `.nvmrc`, `.prettierrc.json`, `README.md`, `astro.config.mjs`, `components.json`, `eslint.config.js`, `node_modules/`, `package-lock.json`, `package.json`, `public/`, `src/`, `supabase/`, `tsconfig.json`, `wrangler.jsonc` (17). Plus 2 conflict siblings (`CLAUDE.md.scaffold`, `.vscode.scaffold`).

## Post-scaffold audit

**Tool**: `npm audit --json`
**Summary**: 0 CRITICAL, 11 HIGH, 7 MODERATE, 0 LOW (18 advisories total)
**Direct vs transitive**: 0/5/1/0 direct of total 0/11/7/0 (npm audit distinguishes direct via `isDirect`)
**Audit exit code**: non-zero (expected when advisories exist — not a halt)

#### CRITICAL findings

None.

#### HIGH findings

Direct (5):
- `@astrojs/cloudflare` — range `<=...-20230912155045 || >=6.3.0`; fix `@astrojs/cloudflare@6.2.4` (semver-major downgrade).
- `@astrojs/react` — range `<=...-20240405144822 || >=3.0.0-beta.0`; fix `@astrojs/react@3.6.2` (semver-major).
- `@tailwindcss/vite` — range `<=4.2.1`; fix available.
- `astro` — range `<=...-20240329190922 || 2.2.0 - 7.0.0-alpha.1`; fix `astro@2.4.5` (semver-major).
- `wrangler` — range `<=...-kickoff-demo || >=3.7.0`; fix `wrangler@3.6.0` (semver-major).

Transitive (6):
- `@cloudflare/vite-plugin` — via miniflare/vite/wrangler/ws; fix available.
- `@vitejs/plugin-react` — via vite; fix `@astrojs/react@3.6.2` (major).
- `devalue` (5.6.3 - 5.8.0) — "Svelte devalue: DoS via sparse array deserialization" (CWE-770, CVSS 7.5); fix available.
- `esbuild` (0.17.0 - 0.28.0) — "Missing binary integrity verification in Deno module enables RCE via NPM_CONFIG_REGISTRY"; fix `astro@2.4.5` (major).
- `vite` (4.2.0-beta.0 - 8.0.3) — via esbuild chain; fix `@astrojs/react@3.6.2` (major).
- `vitefu` — via vite; fix available.

#### MODERATE findings

Direct (1):
- `@astrojs/check` (>=0.9.3) — via @astrojs/language-server; fix `@astrojs/check@0.9.2` (semver-major).

Transitive (6):
- `@astrojs/language-server` (>=2.14.0) — via volar-service-yaml; fix `@astrojs/check@0.9.2` (major).
- `miniflare` — via wrangler chain; fix `wrangler@3.6.0` (major).
- `volar-service-yaml` (<=0.0.70) — fix `@astrojs/check@0.9.2` (major).
- `ws` (8.0.0 - 8.20.0) — "ws: Uninitialized memory disclosure"; fix `wrangler@3.6.0` (major).
- `yaml` (2.0.0 - 2.8.2) — "Stack Overflow via deeply nested YAML collections"; fix `@astrojs/check@0.9.2` (major).
- `yaml-language-server` — fix `@astrojs/check@0.9.2` (major).

#### LOW / INFO findings

None.

**Note**: most fixes are flagged `isSemVerMajor` and resolve to *older* versions — `npm audit fix --force` would downgrade core packages (astro, wrangler, @astrojs/*) and likely break the starter's pinned stack. Treat these as advisory; prefer upstream starter updates over a forced downgrade. No auto-fix was run (out of scope for bootstrapper).

## Hints recorded but not acted on

| Hint                    | Value             |
| ----------------------- | ----------------- |
| bootstrapper_confidence | first-class       |
| quality_override        | false             |
| path_taken              | standard          |
| self_check_answers      | null              |
| team_size               | solo              |
| deployment_target       | cloudflare-pages  |
| ci_provider             | github-actions    |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | true              |
| has_payments            | false             |
| has_realtime            | false             |
| has_ai                  | true              |
| has_background_jobs     | false             |

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` is not needed — this is already a git repo; just `git add` the scaffolded files when ready.
- Review the `.scaffold` siblings the conflict policy created:
  - `CLAUDE.md.scaffold` — the starter's AI-agent guide (commands, architecture, auth flow, conventions). Worth merging useful parts into your own `CLAUDE.md`.
  - `.vscode.scaffold/` — the starter's editor settings/extensions.
- Copy `.env.example` to `.env` (Node) or `.dev.vars` (Cloudflare local dev) and fill `SUPABASE_URL` / `SUPABASE_KEY`.
- Address audit findings per your risk tolerance — the full breakdown is above. Avoid `npm audit fix --force` here; the suggested fixes are major downgrades that would break the pinned stack.
