# coldquill — Build Plan

TDD throughout: write the failing test first, then the code that passes it. Each
task below is sized for one focused agent session.

## T1 — Project scaffold

Files: `wrangler.jsonc`, `package.json`, `tsconfig.json`, `src/worker.ts`,
`public/index.html`, `.dev.vars` (gitignored), `supabase/config.toml`.
Interfaces: `export default { fetch(req, env, ctx) }` returning a static
"not built yet" page.
Test: `test/worker.test.ts` — GET `/` returns 200.
Done: `wrangler dev` serves the page; `npm test` green.

## T2 — Supabase schema + RLS

Files: `supabase/migrations/0001_init.sql` (profiles, personas, sequences,
sequence_emails, `grant_plan_days`, RLS exactly as in `docs/LLD.md`).
Test: migration applies against a local/test Supabase; RLS test queries as user B
and asserts 0 rows for user A's personas/sequences; `unique(sequence_id,
step_number)` confirmed by attempting a duplicate insert.
Done: migration applies cleanly; both tests pass.

## T3 — Local spam-heuristics module

Files: `src/spam.ts` (`scoreSpam(subject, body): {score: number; flags: string[]}`).
Test: `test/spam.test.ts` — table-driven: a clean professional email scores low; an
email stuffed with "FREE!!!", all-caps subject, and a link-shortener domain scores
high with matching flags.
Done: scorer is pure and deterministic, no network/LLM call, fully unit-tested.

## T4 — LLM sequence generator (Anthropic default, DeepSeek override)

Files: `src/providers/anthropic.ts`, `src/providers/deepseek.ts`, `src/sequence.ts`
(`generateSequence(persona, goal, env): Promise<{emails: EmailDraft[]; model:
string; inputTokens: number; outputTokens: number}>`), strict-JSON schema per
`docs/LLD.md`.
Test: `test/sequence.test.ts` — mocked provider responses, asserts 4-step output
shape, asserts malformed/raw-quote JSON is repaired or rejected cleanly (mirrors the
reference product), asserts `resolveProvider(env)` defaults to Anthropic.
Done: both providers unit-tested against mocked fetch, no live API calls in tests.

## T5 — Supa client + handlers (personas, sequences CRUD + generate)

Files: `src/supa.ts`, `src/handlers.ts`, `src/router.ts`.
Interfaces: `route(req, deps): Promise<Response | null>` (same shape as the
reference product), handlers per the route table in `docs/LLD.md`.
Test: `test/handlers.test.ts` — mocked JWT + mocked fetch + mocked LLM, asserts 402
on expired plan/trial, asserts a failed LLM call writes nothing to `sequences`
(502, no partial row), asserts spam scores are attached to each email on insert.
Done: full route set implemented and tested.

## T6 — Regenerate single step

Files: `src/handlers.ts` (`/api/sequences/:id/emails/:step/regenerate`).
Test: `test/regenerate.test.ts` — asserts only the targeted step's row is replaced,
other steps untouched; asserts plan/trial gate applies here too.
Done: regeneration is scoped correctly and tested in isolation from full-sequence
generation.

## T7 — Frontend: personas, generator, sequence view, billing

Files: `public/app.js`, `public/personas.html`, `public/new.html`,
`public/sequence.html`, `public/billing.html`.
Test: none required (static UI); manual check against local `wrangler dev`.
Done: a fresh signup can create a persona, generate a 4-email sequence, see spam
scores per email, and regenerate one step end to end locally.

## T8 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`.
Test: `scripts/smoke.ts` against the deployed Worker — create a persona, generate a
sequence, confirm 4 emails with spam scores are returned, regenerate step 2, confirm
only that step changed.
Done: `wrangler deploy` succeeds; smoke green; secrets set via `wrangler secret
put`; "drafts only, you send it, you're responsible for compliance" disclaimer
visible in the UI; pricing/trial copy on the billing page.
