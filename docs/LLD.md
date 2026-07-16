# coldquill — Low-Level Design

## Architecture

```
Browser (magic-link auth via supabase-js CDN) -- HTTPS --> Cloudflare Worker
  (single Worker: Workers Assets + /api/* routes)
  |-- GET  /            -> static frontend (public/*)
  |-- GET  /config.js   -> {SUPABASE_URL, SUPABASE_ANON_KEY}
  |-- /api/*             -> handlers.ts, JWT verified via GoTrue /auth/v1/user
  v
Supabase Postgres (RLS) + GoTrue auth        Anthropic (claude-sonnet-4-6, default)
  personas, sequences, sequence_emails           or DeepSeek (cost override)
  Worker writes via service-role key             via plain fetch, strict JSON
                                              Local spam-heuristics module (no
                                                external call — in-Worker regex/scoring)
```

Request flow: browser POSTs `/api/sequences {persona_id, goal}` -> Worker checks the
plan/trial gate -> loads the persona -> calls the LLM once for the whole sequence
(single request, strict JSON output) -> runs each generated email body through the
local spam-heuristics scorer -> inserts the sequence and its emails -> returns the
full draft with per-email spam scores. Regenerating a single step re-runs the same
path scoped to one email.

## Data model

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  plan_active_until timestamptz,
  trial_ends_at timestamptz not null default (now() + interval '3 days'),
  created_at timestamptz not null default now()
);
alter table public.profiles enable row level security;
create policy "read own profile" on public.profiles for select using (auth.uid() = user_id);

create table public.personas (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,                -- "SaaS founders, seed stage"
  product_pitch text not null,
  icp_notes text,
  tone text not null default 'professional',
  created_at timestamptz not null default now()
);
alter table public.personas enable row level security;
create policy "owner rw personas" on public.personas for all using (auth.uid() = user_id);

create table public.sequences (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  persona_id uuid not null references public.personas(id),
  title text not null,
  goal text,
  status text not null default 'draft' check (status in ('draft','final')),
  model text,
  input_tokens int,
  output_tokens int,
  created_at timestamptz not null default now()
);
alter table public.sequences enable row level security;
create policy "owner rw sequences" on public.sequences for all using (auth.uid() = user_id);

create table public.sequence_emails (
  id uuid primary key default gen_random_uuid(),
  sequence_id uuid not null references public.sequences(id) on delete cascade,
  step_number int not null,
  subject text not null,
  body text not null,
  spam_score int not null default 0,     -- 0-100, higher = riskier
  spam_flags jsonb not null default '[]',
  created_at timestamptz not null default now(),
  unique (sequence_id, step_number)
);
alter table public.sequence_emails enable row level security;
create policy "owner rw sequence_emails" on public.sequence_emails for all using (
  sequence_id in (select id from public.sequences where user_id = auth.uid())
);

create or replace function public.grant_plan_days(p_user uuid, p_days int) returns void
language plpgsql security definer set search_path = public as $$
begin
  update profiles set plan_active_until =
    greatest(now(), coalesce(plan_active_until, now())) + (p_days || ' days')::interval
  where user_id = p_user;
end $$;
revoke execute on function public.grant_plan_days(uuid, int) from public, anon, authenticated;
grant execute on function public.grant_plan_days(uuid, int) to service_role;
```

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/personas` | GET/POST | JWT | List / create a persona | 402 if plan/trial expired |
| `/api/personas/:id` | PATCH | JWT | Edit a persona | 404 |
| `/api/sequences` | POST | JWT | Generate a new draft sequence (LLM call) | 402, 502 on LLM failure |
| `/api/sequences` | GET | JWT | List caller's sequences | — |
| `/api/sequences/:id` | GET | JWT | Fetch one sequence + its emails | 404 |
| `/api/sequences/:id/emails/:step` | PATCH | JWT | Manual edit of one email | 404 |
| `/api/sequences/:id/emails/:step/regenerate` | POST | JWT | Re-run generation for one step | 402, 502 |
| `/api/me` | GET | JWT | Profile + plan/trial status | — |

## LLM strategy

**Provider: Anthropic (`claude-sonnet-4-6`) by default**, `LLM_PROVIDER=deepseek`
override available — copywriting quality is the core product value here (unlike
leadlime's extraction task), so the default favors the stronger model even at higher
per-call cost, since this is a monthly-plan product where marginal LLM cost is
amortized across a subscriber's usage rather than charged per call.

Prompt strategy: system prompt describes the persona (product pitch, ICP, tone) and
the sequence goal, instructs a 4-step structure (intro -> value-add follow-up ->
social-proof/case-study -> breakup), strict JSON output, same "no raw quotes inside
JSON strings" robustness rule as the shipped product's prompt. Output schema sketch:

```json
{
  "emails": [
    {"step": 1, "subject": "string", "body": "string"},
    {"step": 2, "subject": "string", "body": "string"},
    {"step": 3, "subject": "string", "body": "string"},
    {"step": 4, "subject": "string", "body": "string"}
  ]
}
```

Cost estimate: ~1-2k input tokens (persona + prompt) + ~1.2-1.8k output tokens (4
emails) per sequence generation. At roughly $3/M input, $15/M output list pricing for
Claude Sonnet (verify current rates before build), that's approximately
$0.02-0.03 per sequence — well inside a Rs499/mo subscription's margin even at heavy
usage. Regenerating a single step is a small fraction of that.

**Sync vs async: SYNC.** One LLM call generating ~4 short emails completes in single-
digit to low-tens of seconds — nowhere near Cloudflare's ~100s cap. No VPS/async
dispatch needed; this is a much smaller generation task than the reference product's
whole-PDF contract review.

## Frontend pages

`/` login · `/personas` persona CRUD · `/new` sequence generator (pick persona +
goal) · `/sequence/:id` draft view with per-email spam score, edit, and regenerate
controls · `/billing` plan status + manual UPI + Razorpay checkout (post-KYC).

## Error handling + credits/refund flow

No credit spend/refund cycle — monthly-plan gate like seatsaver/billplume/remindloop.
Every mutating route checks `plan_active_until > now() OR trial_ends_at > now()`
before writing, else 402 with an upgrade URL. If the LLM call fails or returns
malformed JSON after one repair attempt, the Worker returns 502 and writes nothing —
no half-generated sequence is ever persisted.

## Integrations and launch gates

No sending integration in v1 by design (see Risks) — nothing to gate on an ESP
partnership or deliverability review. Razorpay needs KYC before checkout goes live;
manual UPI is the day-1 fallback, same as the rest of this batch.

## Security notes

RLS scopes every table to `auth.uid()`; the Worker's service-role key is the only
writer. Persona fields (`product_pitch`, `icp_notes`) are user-controlled free text
fed into the LLM prompt — length-capped server-side before the call; low
prompt-injection risk since output only ever renders back to the same authenticated
user, but the JSON-schema constraint plus output-length caps still apply as
defense-in-depth. Secrets (`SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY` or
`DEEPSEEK_API_KEYS`) via `wrangler secret put`; plain vars for the Supabase URL and
anon key.

## Out of scope for v1

Actual sending (SMTP/ESP integration, warmup, SPF/DKIM/DMARC), deliverability
monitoring, A/B testing of subject lines, CRM sync, LinkedIn/multi-channel
sequences, multi-language generation, team accounts, LLM-based (rather than local
heuristic) spam scoring.
