---
project: "10xCards"
version: 1
status: draft
created: 2026-06-23
updated: 2026-06-23
prd_version: 1
main_goal: speed
top_blocker: time
---

# Roadmap: 10xCards

> Derived from `context/foundation/prd.md` (v1) + `tech-stack.md` + `infrastructure.md` + auto-researched codebase baseline.
> Edit-in-place; archive when superseded.
> Slices below are listed in dependency order. The "At a glance" table is the index.

## Vision recap

Authoring flashcards by hand is the friction that kills adoption of spaced repetition: existing tools repeat cards well but still force the learner to create them manually. 10xCards removes that step — paste source text, let AI propose question/answer candidates, accept the good ones, and review them on a ready-made spaced-repetition schedule. The product wedge — the one trait that, if removed, makes this indistinguishable from a generic flashcard app — is that cards are AI-generated from the learner's own pasted text **and** human-gated before they enter the deck.

## North star

**S-02: srs-review-session** — user studies their saved cards through the integrated spaced-repetition algorithm. This is the chosen validation milestone: it proves the learning loop actually closes (a card created → reviewed → rescheduled), which is what turns 10xCards from a generator into a learning tool.

> "North star" here means the smallest end-to-end slice whose successful delivery proves the core product hypothesis — placed as early as Prerequisites allow because everything else only matters if this works. Note: review needs a populated deck, so minimal manual card creation (S-01) is the unavoidable predecessor; the north star is sequenced immediately after it.

## At a glance

| ID    | Change ID                | Outcome (user can …)                                              | Prerequisites | PRD refs               | Status   |
| ----- | ------------------------ | ---------------------------------------------------------------- | ------------- | ---------------------- | -------- |
| F-01  | flashcards-data-layer    | (foundation) per-user flashcard storage with hard data isolation | —             | Access Control, NFR    | ready    |
| S-01  | manual-create-and-list   | create a card by hand and see it in their list                   | F-01          | FR-005, FR-006         | proposed |
| S-02  | srs-review-session       | study due cards via the spaced-repetition algorithm              | S-01          | US-02, FR-009          | proposed |
| S-03  | ai-gated-generation      | paste text, review AI candidates, accept/edit/reject into deck   | F-01, S-01    | US-01, FR-003, FR-004  | blocked  |
| S-04  | edit-and-delete-cards    | edit and delete a saved card                                     | S-01          | FR-007, FR-008         | proposed |

## Streams

Navigation aid — groups items that share a Prerequisites chain. Canonical ordering still lives in the dependency graph below; this table is the proposed reading order across parallel tracks.

| Stream | Theme                  | Chain                          | Note                                                                 |
| ------ | ---------------------- | ------------------------------ | ------------------------------------------------------------------- |
| A      | Talia → pętla nauki    | `F-01` → `S-01` → `S-02`       | Critical path to the north star; must-have spine under `speed`.      |
| B      | Generowanie AI (wedge) | `S-03`                         | Joins Stream A at `S-01` (reuses the card-save path); blocked on OQ#1. |
| C      | Zarządzanie fiszkami   | `S-04`                         | Joins Stream A at `S-01`; lower-risk management work.                |

## Baseline

What's already in place in the codebase as of `2026-06-23` (auto-researched + user-confirmed).
Foundations below assume these are present and do NOT re-scaffold them.

- **Frontend:** present — Astro 6 + React 19 islands + Tailwind 4 + shadcn/ui (`src/components/ui/button.tsx`, `src/components/auth/*`, `src/layouts/Layout.astro`).
- **Backend / API:** partial — Astro API routes exist only for auth (`src/pages/api/auth/{signin,signout,signup}.ts`); no flashcard or generation endpoints yet.
- **Data:** absent — Supabase client wired (`src/lib/supabase.ts`) but `supabase/` holds only `config.toml` — no migrations, no tables.
- **Auth:** present — Supabase cookie SSR + `src/middleware.ts` + signin/signup/confirm-email screens + `/dashboard` protected via `PROTECTED_ROUTES`. **This satisfies FR-001 and FR-002**, so no auth slice exists below.
- **Deploy / infra:** present — Cloudflare Workers (`@astrojs/cloudflare`, `wrangler`), documented in `infrastructure.md`.
- **Observability:** absent — no logging/Sentry/OTel deps. The "source text leaves no trace" NFR is handled by *not* logging request bodies (in S-03), not by an observability foundation.

## Foundations

### F-01: Per-user flashcard data layer

- **Outcome:** (foundation) a `flashcards` table exists with row-level security so each authenticated user can read/write only their own cards.
- **Change ID:** flashcards-data-layer
- **PRD refs:** Access Control (flat user model, hard data isolation), Guardrails ("a user never sees another user's flashcards"), enables FR-005..FR-009.
- **Unlocks:** S-01 (create/list), S-02 (review), S-03 (generation save), S-04 (edit/delete) — every card slice writes or reads through this layer.
- **Prerequisites:** — (auth is already present per Baseline).
- **Parallel with:** —
- **Blockers:** —
- **Unknowns:** —
- **Risk:** The data-isolation guardrail must be enforced at the DB layer (RLS), not just app code — getting it wrong leaks one user's cards to another. Kept minimal (base card table + ownership/RLS); SRS scheduling columns are added additively by S-02, so this stays the smallest enabler and is not a "build the whole data layer" project.
- **Status:** ready

## Slices

### S-01: Manual card creation and listing

- **Outcome:** user can manually create a flashcard (question + answer) and see it in their list of saved cards.
- **Change ID:** manual-create-and-list
- **PRD refs:** FR-005, FR-006
- **Prerequisites:** F-01
- **Parallel with:** —
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Establishes the card read/write path the whole deck reuses; kept to the smallest user-visible capability (create + list) so the north-star review loop is reachable fast. The manual path is the PRD's deliberate fallback, not a competitor to the AI default.
- **Status:** proposed

### S-02: Spaced-repetition review session  ← north star

- **Outcome:** user can start a review session and study their due cards; the integrated algorithm decides order and the next interval from the user's response; an empty/no-cards-due state shows an explanatory message.
- **Change ID:** srs-review-session
- **PRD refs:** US-02, FR-009
- **Prerequisites:** S-01
- **Parallel with:** S-03, S-04
- **Blockers:** —
- **Unknowns:** —
- **Risk:** The north star — proves the learning loop closes. Integrates a ready-made SRS algorithm (Non-Goal: no custom engine), so the app must not override its scheduling, and the no-due/empty state must degrade gracefully rather than break. Adds SRS scheduling columns to F-01's table additively.
- **Status:** proposed

### S-03: AI generation with acceptance gate

- **Outcome:** user can paste source text, receive AI-generated candidate cards with continuous progress feedback, and accept / edit / reject each one before any is saved to the deck.
- **Change ID:** ai-gated-generation
- **PRD refs:** US-01, FR-003, FR-004
- **Prerequisites:** F-01, S-01
- **Parallel with:** S-02, S-04
- **Blockers:** —
- **Unknowns:**
  - Maximum length of pasted text accepted for generation (PRD OQ#1) — Owner: user. Block: yes.
- **Risk:** The product wedge, but blocked on the input-length decision: per `infrastructure.md`, unbounded input is the main failure mode (Worker CPU/wall + subrequest limits), and the "source text leaves no trace" NFR means request bodies must never be logged. Sequenced after the chosen north star (review loop) even though it validates success criterion #1 (≥75% acceptance) — resolving OQ#1 promotes it to `ready`.
- **Status:** blocked

### S-04: Edit and delete saved cards

- **Outcome:** user can edit a saved flashcard and delete one.
- **Change ID:** edit-and-delete-cards
- **PRD refs:** FR-007, FR-008
- **Prerequisites:** S-01
- **Parallel with:** S-02, S-03
- **Blockers:** —
- **Unknowns:**
  - Is deletion irreversible or soft-delete/undo (PRD OQ#2)? — Owner: user. Block: no (defaults to hard delete).
- **Risk:** Lower-risk management slice over the existing card path. Delete semantics are an open decision but a hard-delete default keeps it unblocked; post-save editing is required because errors surface during review (FR-007 rationale).
- **Status:** proposed

## Backlog Handoff

| Roadmap ID | Change ID                | Suggested issue title                                  | Ready for `/10x-plan` | Notes                                      |
| ---------- | ------------------------ | ----------------------------------------------------- | --------------------- | ------------------------------------------ |
| F-01       | flashcards-data-layer    | Per-user flashcard storage with RLS                   | yes                   | Run `/10x-plan flashcards-data-layer`       |
| S-01       | manual-create-and-list   | Manually create a card and list saved cards           | no                    | Needs F-01 done                            |
| S-02       | srs-review-session       | Spaced-repetition review session (north star)         | no                    | Needs S-01 done                            |
| S-03       | ai-gated-generation      | AI generation with accept/edit/reject gate            | no                    | Blocked on OQ#1 (max input length) + S-01  |
| S-04       | edit-and-delete-cards    | Edit and delete saved cards                           | no                    | Needs S-01 done                            |

## Open Roadmap Questions

1. **What is the maximum length of pasted text accepted for AI generation?** — Owner: user. Block: `S-03`. Load-bearing: gates the wedge slice and the ≥75%-acceptance success criterion; `infrastructure.md` ties Worker CPU/subrequest failures to the missing bound.
2. **Is flashcard deletion irreversible, or soft-delete / undo?** — Owner: user. Block: none (`S-04` defaults to hard delete).
3. **Does the flashcard list need search / filtering, and at what card count?** — Owner: user. Block: none (out of MVP scope — see Parked).
4. **What is the session model (persistence / "remember me" / logout-all-devices)?** — Owner: user. Block: none (downstream of the already-present auth layer).

## Parked

- **Custom advanced SRS algorithm** — Why parked: PRD Non-Goals; 10xCards integrates a ready-made SRS engine, not its own.
- **Multi-format import (PDF, DOCX, …)** — Why parked: PRD Non-Goals; input is pasted text only in v1.
- **Deck sharing between users** — Why parked: PRD Non-Goals; reinforces the single-user flat access model.
- **Mobile apps / third-party edu integrations** — Why parked: PRD Non-Goals; web-only MVP.
- **List search / filtering** — Why parked: PRD OQ#3; out of MVP scope, deferred until card counts make a plain list painful.

## Done

(Empty on first generation. `/10x-archive` appends entries here when a change whose Change ID matches a roadmap item is archived.)
