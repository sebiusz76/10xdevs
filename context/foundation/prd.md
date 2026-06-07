---
project: "10xCards"
version: 1
status: draft
created: 2026-06-07
context_type: greenfield
product_type: web-app
target_scale:
  users: medium
  qps: low
  data_volume: small
timeline_budget:
  mvp_weeks: 3
  hard_deadline: null
  after_hours_only: true
---

# 10xCards — Product Requirements Document

## Vision & Problem Statement

Creating high-quality educational flashcards by hand is time-consuming. The friction of authoring cards — rephrasing source material into clean question/answer pairs — interrupts the learning rhythm and discourages people from adopting spaced repetition, even though it is one of the most effective study methods.

The insight: existing tools (Anki, Quizlet) are excellent at *repeating* cards but still force the learner to *create* them manually. That creation step is where adoption dies. Removing it with AI generation from pasted text turns spaced repetition from a high-effort commitment into a low-effort default.

## User & Persona

**Primary persona — the self-taught / professional learner.** An adult who studies after hours (e.g. in technical fields) from articles, documentation, and books. They already have source material in hand; what they lack is the time and patience to convert it into well-formed flashcards. They want to paste what they're reading and start reviewing — not build and maintain a deck by hand.

## Success Criteria

### Primary
- At least 75% of AI-generated flashcards are accepted by the user (a proxy for generation quality).
- At least 75% of all flashcards a user owns are created via AI generation rather than manually (a proxy for the AI path being the default, not a fallback).
- The end-to-end flow works: sign up → paste text → AI generates candidates → user accepts/edits/rejects → accepted cards saved → user reviews them through the spaced-repetition algorithm.

### Secondary
- Users return to review sessions on subsequent days (retention) — a signal that studying actually became a habit, not just a one-off generation.

### Guardrails
- Pasted source text does not leak and is not used beyond serving the user's own generation request (privacy of input).
- A user never sees another user's flashcards — hard data isolation following from the flat account model.
- No AI-generated card enters a user's deck without explicit acceptance — no auto-save of raw candidates; the user is always in control.
- Generation gives continuous visible feedback and does not freeze the interface, even when processing takes longer.

## User Stories

### US-01: User generates flashcards from pasted text

- **Given** a logged-in user with source material to study
- **When** they paste the text and request generation
- **Then** they receive a set of candidate flashcards and can accept, edit, or reject each one before any is saved

#### Acceptance Criteria
- No candidate is saved to the deck until the user explicitly accepts it.
- Editing a candidate before acceptance saves the edited version, not the original.
- Rejected candidates are discarded and do not reappear in the same set.
- The user sees continuous feedback while generation is in progress.

### US-02: User reviews flashcards via spaced repetition

- **Given** a logged-in user with at least one saved flashcard due for review
- **When** they start a review session
- **Then** the integrated spaced-repetition algorithm presents cards in the order it determines and schedules the next review based on their response

#### Acceptance Criteria
- Only the user's own flashcards appear in their session.
- The algorithm determines which cards are due and the next review interval — the app does not override its scheduling.
- An empty/no-cards-due state shows an explanatory message, not a broken session.

## Functional Requirements

### Accounts
- FR-001: User can register an account with email and password. Priority: must-have
  > Socrates: Counter-argument considered: "managing passwords (hashing, reset, security) is costly; OAuth could be cheaper/safer." Resolution: kept as-is; email+password is the simplest known model for durable storage. OAuth is a downstream stack choice, not a product decision.
- FR-002: User can log in and log out. Priority: must-have
  > Socrates: No counter-argument; it stands as written. (Session persistence / "remember me" / logout-all-devices noted as an Open Question, not part of this FR.)

### AI generation
- FR-003: User can paste text and request AI-generated flashcard candidates. Priority: must-have
  > Socrates: Counter-argument considered: "no length bound means generation can be costly or produce weak cards from very long blocks." Resolution: kept; the FR stands but an input-length bound is needed — routed to Open Questions.
- FR-004: User can review each AI candidate and accept, edit, or reject it before it is saved. Priority: must-have
  > Socrates: Counter-argument considered: "high-confidence cards could be auto-accepted to reduce clicks." Resolution: kept; explicit acceptance is a guardrail (user is always in control, no auto-save of raw candidates). Auto-accept would violate that guardrail.

### Manual creation
- FR-005: User can manually create a flashcard with a question and an answer. Priority: must-have
  > Socrates: Counter-argument considered: "manual creation contradicts the AI-first thesis and could be cut from v1." Resolution: kept as a fallback — for when AI fails or for short additions; it does not compete with the AI path being the default.

### Flashcard management
- FR-006: User can view a list of their saved flashcards. Priority: must-have
  > Socrates: Counter-argument considered: "a plain list without search/filter won't scale." Resolution: kept; search/filter is out of MVP scope and routed to Open Questions for a later version.
- FR-007: User can edit a saved flashcard. Priority: must-have
  > Socrates: Counter-argument considered: "editing at acceptance time (FR-004) makes post-save editing redundant." Resolution: kept; typos and errors often surface during review sessions, so post-save editing is required.
- FR-008: User can delete a saved flashcard. Priority: must-have
  > Socrates: Counter-argument considered: "hard delete vs trash/undo." Resolution: kept; whether deletion is irreversible or soft-delete with undo is routed to Open Questions.

### Review
- FR-009: User can study their flashcards through an integrated spaced-repetition algorithm. Priority: must-have
  > Socrates: Counter-argument considered: "MVP could end at generation+save and add review in v2." Resolution: kept — review is the core of the product. Without it, 10xCards is a flashcard generator, not a learning tool, and the Primary success criteria (which require reviewing through the algorithm) would no longer hold.

## Non-Functional Requirements

- Generation gives acknowledgement within a few seconds and continuous visible progress for any operation that takes longer than two seconds; the interface never freezes while processing.
- Source text submitted for AI generation leaves no trace in operator-accessible storage after the request that consumed it completes.
- A failed login does not lock out a legitimate user who mistypes their password a few times in a row, but credential-stuffing at scale is rejected before reaching the auth check.
- The product remains usable on the latest two major versions of the mainstream desktop browsers (web-only MVP).

## Business Logic

Given a block of source text, the application decides what is worth learning and transforms it into well-formed question/answer flashcard pairs that the learner can review.

The rule consumes a single user-facing input: a block of text the learner pastes in (e.g. an article, documentation, or book excerpt). Its output is a set of candidate flashcards, each a self-contained question paired with its answer. The learner encounters this output as a review list where each candidate can be accepted, edited, or rejected before anything is saved — the application proposes, the learner disposes. Accepted cards then enter the learner's deck and become subject to the spaced-repetition schedule.

## Access Control

Email + password accounts. Flat user model — there is a single user type; every authenticated user can create, view, edit, and delete only their own flashcards. No roles, no admin tier, no sharing between users in the MVP. An unauthenticated visitor cannot reach any flashcard data and is directed to sign in / sign up.

## Non-Goals

- **No custom advanced spaced-repetition algorithm** — 10xCards integrates a ready-made SRS algorithm rather than building its own (no SuperMemo/Anki-style engine). Buy-vs-build resolved in favor of integration.
- **No multi-format import (PDF, DOCX, etc.)** — input is pasted text only; file parsing is out of scope for v1.
- **No deck sharing between users** — no sharing of flashcard sets; reinforces the single-user, flat access model.
- **No mobile apps and no integrations with other educational platforms** — web-only MVP; native mobile and third-party edu integrations are out of scope.

## Open Questions

1. **What is the maximum length of pasted text accepted for AI generation?** — Owner: user. Surfaced by FR-003 Socrates round; affects cost and card quality. Needs a defined bound.
2. **Is flashcard deletion irreversible, or is there a soft-delete / undo?** — Owner: user. Surfaced by FR-008 Socrates round.
3. **Does the flashcard list need search / filtering, and at what card count?** — Owner: user. Surfaced by FR-006; out of MVP scope but should be confirmed as a deliberate non-goal.
4. **What is the session model (persistence / "remember me" / logout-all-devices)?** — Owner: user. Surfaced by FR-002; minor, downstream of auth implementation.
