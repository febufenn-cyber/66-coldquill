# coldquill

Cold-email sequence writer with spam-trigger checking and persona memory.

**Status: planned — not yet built (50-SaaS challenge #66)**

## The problem

B2B founders and SDRs rewrite the same cold-email sequence structure for every new
campaign, guessing at what will trip spam filters, and re-explaining their product
and ICP to whatever tool they're using each time.

## Target buyer

B2B founders and SDRs running their own outbound campaigns who want a fast first
draft of a multi-step sequence, not a full sending platform.

## Pricing hypothesis

Rs499/mo, flat — unlimited sequence generations. 3-day free trial on signup.

## Stack

Cloudflare Worker (TypeScript, assets + `/api/*` in one Worker) fronting Supabase
(Postgres + RLS, magic-link auth). LLM layer: Anthropic (`claude-sonnet-4-6`)
default for copywriting quality, provider-switchable to DeepSeek for cost. Spam
checking is local, deterministic heuristics — not a model call. Monetization is a
monthly plan gate, matching seatsaver/billplume/remindloop in this batch.

## How to continue this build

Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the
ordered build tasks. `CLAUDE.md` points an agent at the shared conventions in the
private reference repo.

## Risks / constraints

- **Drafts only — v1 does not send email.** Sending means deliverability,
  authentication (SPF/DKIM/DMARC), warmup, and ESP-relationship management — a
  separate rabbit hole this product deliberately stays out of. Users copy drafts
  into their own sender.
- **Spam-trigger checking runs locally** (word-list + pattern heuristics in the
  Worker), not via an LLM call — cheap, fast, and doesn't depend on a model's
  training data being current on filter behavior.
- **Not a compliance tool.** The user is responsible for consent, unsubscribe
  handling, and applicable law (CAN-SPAM, India's IT Act, etc.) for whatever they
  send — this must be disclaimed in the UI.
- Persona memory (product pitch, ICP, tone) is stored per user and reused across
  sequences — see LLD data model.
