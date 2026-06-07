---
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
---

## Why this stack

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
