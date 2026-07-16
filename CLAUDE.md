This repo is product #66 of Febin's 50-SaaS challenge. Nothing is built yet —
docs/ is the source of truth. Read docs/LLD.md then docs/PLAN.md and execute tasks
in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo
febufenn-cyber/50-saas (contract-reviewer/) — same Worker+Supabase+RLS+provider
patterns, adapted here to a monthly plan gate instead of per-op credits (see LLD).

The drafts-only (no sending) scope, the local (non-LLM) spam heuristics, and the
Anthropic-default provider choice in the LLD are verified decisions — do not
re-litigate without evidence.
