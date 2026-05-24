<!--
============================================================
THIS FILE IS NOT PART OF THE ABN APPLICATION.
============================================================
PURPOSE: Session memory for Jacob + Claude (claude.ai chat).
Used so that when Jacob starts a new Claude chat, he pastes
the raw URL and Claude instantly knows where we are.

RAW URL (paste in new Claude chat):
https://raw.githubusercontent.com/abn-systems/abn-session-memory/main/JACOB_SESSION.md

THIS FILE HAS ZERO IMPACT ON:
- Backend code or tests
- Frontend or dashboard
- Deployments or CI/CD
- Any ABN system functionality

DO NOT modify this file as part of any feature or bug fix.
DO NOT mix this with ABN application memory or logic.

CLAUDE CODE RULES FOR THIS FILE:
1. After every completed batch: update this file with what was built
2. After every completed batch: update CHAT_LOG.md with decisions made
3. Then run: git add JACOB_SESSION.md CHAT_LOG.md
4. Then run: git commit -m "chore: update session memory after [batch name]"
5. Then run: git push
6. This triggers auto-sync to public mirror at abn-session-memory repo
7. Jacob uses the raw URL in new chats — never loses context

WHY THIS EXISTS:
Claude.ai chat has a file limit. When the chat gets too long,
Jacob opens a new chat. Without this file, all context is lost.
This file prevents that. It is the single source of truth for
what has been built, what prompts were given, and what is next.
============================================================
-->

# ABN — Session Memory
Last updated: 2026-05-24
Repo: https://github.com/abn-systems/ABN
Raw URL: https://raw.githubusercontent.com/abn-systems/ABN/main/JACOB_SESSION.md

## How Jacob uses this file
In any new Claude chat, paste this URL and say "read this":
https://raw.githubusercontent.com/abn-systems/ABN/main/JACOB_SESSION.md

## What is built (as of 2026-05-24)
- Backend: **962 tests passing** (939 V1 baseline + 23 added in Batch 1), V1 feature-complete
- Frontend: Tauri desktop dashboard live, Swedish UI, NSIS installer built (`ABN_1.0.0_x64-setup.exe`)
- Landing: 33 prerendered static pages live on Vercel (abn-nine.vercel.app), full dark theme
- Auth: Clerk + RBAC (NODE_ADMIN / AGENT_MANAGER / VIEWER / AUDITOR) + named-employee invite flow live
- AgentDetailPage.tsx: built and **fully hydrated** by the new Finding model + 3 endpoints (Batch 1)
- Post-commit + post-merge git hooks: auto-create Jacob-authored deploy commits so Vercel Hobby plan deploys every push

## Batches done
- Sessions 1-5: Full backend, auth, landing, Tauri dashboard, legal centre, all solution / company pages, mega-menu nav, dark footer, /api, /transparency, /pricing, /changelog, /status — locked in as of session 5
- **Batch 1 (DONE 2026-05-24)** — Finding model + AgentDetailPage hydration

## Batch 1 — Finding model + API endpoints (DONE 2026-05-24)
**Prompt given:** Add Finding model, Alembic migration, wire into OPERA, GET /api/agents/{id}/findings, /trend, /runs/{run_id}/verification, surface tools_used + insight_layer on GET /api/agents/{id}. 15+ tests. All 939 existing must pass.

**Result:**
- New `findings` table in `backend/database/models.py` (id UUID, run_id, agent_id, tenant_id, title, description, severity, impact_eur, delta_pct, attested, source_system, field_name, agent_value, source_value, created_at).
- Alembic revision `c4f1ad9e8a72_add_findings_table.py` (depends on `88e0d47ce0ae`).
- Single source of truth `backend/agent_runtime/findings_persistence.py::save_findings_from_output` — idempotent per `run_id`, called from both `OPERARunner._save_run_record` and AAEA `executor.execute` post-ESE so the same helper covers both paths.
- New endpoints in `backend/api/routes/agents.py`:
  - `GET /api/agents/{id}/findings?run_id=&since=&limit=` — latest run by default, optional run pin / date filter
  - `GET /api/agents/{id}/trend?months=N` — per-month aggregation (impact_eur sum, findings count, avg confidence, runs count)
  - `GET /api/agents/{id}/runs/{run_id}/verification` — RAL field-level attestation from `abn_attestations`
- `GET /api/agents/{id}` extended with `tools_used: list[str]` (from blueprint) and `insight_layer: dict | None` (AGP — stored value wins, derived via `select_insight_layer` otherwise).
- 23 new tests in `backend/tests/test_findings_api.py` covering helper idempotency, severity normalisation, string-numeric coercion, OPERA wire-up, all three endpoints' happy paths + edge cases + 4xx validation, get_agent extensions.
- Full backend suite: **962 passed**, 0 failed. No regressions.
- CLAUDE.md "Known issues" entry for the missing `findings` table flipped open → resolved.

## Batch 2 — NEXT
CI/CD pipeline (GitHub Actions) — automated tests on every push. The repo already has `.github/workflows/ci.yml` from session 1; Batch 2 audits it, adds branch protection, adds the landing build job, wires test coverage reporting.

## Batch 3
Stripe payments — Professional plan (€299/month).

## Batch 4
www.abnplatform.com DNS + real public API endpoint (currently the API is described on `/api` but no real api.abnplatform.com host exists yet).

## Batch 5
Fortnox end-to-end (needs org number + API key from Jacob).

## Blockers
- Fortnox: waiting for org number + API key from Jacob
- DNS: Proton mail propagation (hello@abnplatform.com)
- Google Cloud OAuth: waiting for mail-domain verification

## Technical debt
- 7 duplicated package dirs (`agent_evolution/agent_evolution/`, `consistency/consistency/`, `delivery/delivery/`, `gdpr/gdpr/`, `inquiry_engine/inquiry_engine/`, `onboarding/onboarding/`, `opera/opera/`, `output_schema/output_schema/`) — cleanup sweep deferred
- Two OPERA runners coexist (`agent_engine/opera/runner.py` production, `opera/integrated_runner.py` legacy) — consolidation deferred
- `AgentBlackboard` table missing — reads degrade gracefully to empty list
- Blueprint version history missing — `POST /api/agents/{id}/rollback/{v}` returns 501
- `services/abn-security/` (Go) not compiled in this environment — CI job covers it
- Many modules use deprecated `datetime.utcnow()` — cosmetic warnings only

## Rules
- Always read CLAUDE.md first
- Grep before building — no duplication (Engineering rule #1 in CLAUDE.md)
- All prompts to Claude Code in English; code/comments/identifiers in English (rule #3)
- Act as 4 engineers: Architect, Engineer, Reviewer, Optimizer
- One batch at a time, green build before next (rule #8)
- After every batch: update this file + CHAT_LOG.md and push to GitHub (see `## Session Management` in CLAUDE.md)
