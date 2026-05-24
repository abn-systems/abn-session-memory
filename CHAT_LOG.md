<!--
============================================================
THIS FILE IS NOT PART OF THE ABN APPLICATION.
============================================================
PURPOSE: Log of decisions made between Jacob and Claude chat.
Updated manually when Jacob asks Claude to update it.
Has zero impact on any ABN code, tests, or deployment.
============================================================
-->

# ABN — Chat History (Jacob + Claude)
This file is updated when Jacob asks Claude to update it.

## 2026-05-24 — Batch 6 — Stripe billing
- Engineering rule #1: greps confirmed zero existing Stripe code (only an unrelated YAML connector name `"stripe"` in `test_delivery.py`). Pure additions.
- Caught FIVE bugs in the spec before writing any code:
  1. `from core.database import get_db` — module doesn't exist; real path is `database.session`. Routes throughout the repo use `database.session.get_db`.
  2. `from core.logging import get_logger` — module doesn't exist; codebase uses stdlib `logging.getLogger` directly.
  3. `STRIPE_*: str = ""` UPPER_CASE field names — Settings convention is lowercase fields; `case_sensitive=False` already maps UPPER_CASE env vars onto them, so the .env contract still uses `STRIPE_SECRET_KEY=…`.
  4. `stripe.error.SignatureVerificationError` — alias namespace gone in stripe-python 8.x; canonical name is `stripe.SignatureVerificationError`.
  5. `invoice.paid` stored `data.get("customer", "")` (the Stripe customer ID) in the `tenant_id` column. Tenant ID must be looked up via the Subscription row's `stripe_customer_id`.
- One frontend bug too: spec called `api.get('/api/billing/...')` but the axios wrapper's baseURL already contains `/api`. Fixed by dropping the prefix to `'/billing/...'`.
- All four billing routes refuse to call live Stripe when `stripe_secret_key` is empty (503). Webhook refuses without the signature header or webhook secret. Engineering rule #6 — fail-closed everywhere.
- `invoice.paid` is idempotent on `stripe_invoice_id`; `checkout.session.completed` is upsert-by-tenant; `customer.subscription.*` is upsert-by-stripe_subscription_id. Stripe retries cannot multiply rows. Engineering rule #7.
- 20 new backend tests in `tests/test_billing_api.py`, all Stripe SDK calls mocked. Full backend suite: 1010 passed (990 + 20). Frontend: typecheck clean, 60 tests green, Vite build green.

## 2026-05-24 — Batch 5 — ConnectorsPage live-feed
- Engineering rule #1: three greps confirmed nothing existed for the new endpoint, model name, or component. Pure additions, no rewrites.
- Backend: new `GET /api/connectors/live-feed` declared above `/{connector_id}` so FastAPI literal-match wins. Regression test included so a future refactor can't accidentally re-order them and reintroduce the int-parse 422.
- Frontend: used the existing axios `api` wrapper, not raw `fetch` from the spec — Engineering rule #2 (one HTTP idiom across `client.ts`). `getLiveFeed` returns `[]` on failure so a fresh-install Node never paints a red banner.
- ConnectorsPage: extended imports + added `<LiveFeed />` invocation. The new component lives as a sibling function in the same file; the existing `ConnectorsPage` body is untouched (rule "do not rewrite").
- Surface contract: chip + event_type + pseudonymised case_key + relative timestamp. No raw values, no amounts, no PII. The footer reaffirms "Inget data lämnar din miljö".
- Full backend suite: 990 passed (982 + 8). Frontend: typecheck clean, 60 tests green, Vite build green.

## 2026-05-24 — Batch 4 — Multi-platform Tauri build + download page redesign
- Engineering rule #1: ran the user's three greps to confirm zero drift since the audit. `tauri.conf.json` still on `["nsis"]`; no `build-release.yml`; download page still 236 lines. Nothing pre-existing to extend — pure greenfield.
- Plan A (build workflow): caught two issues in the spec before writing code:
  (1) `dtolnay/rust-toolchain`'s `targets:` field expects raw triples, but the spec piped `--target X` flag strings into it — every cell would have failed at the toolchain install step. Split the matrix into `target:` (for the action) and `args:` (for tauri-action).
  (2) macOS Universal needs BOTH `x86_64-apple-darwin` and `aarch64-apple-darwin` installed so Tauri's `--target universal-apple-darwin` can lipo-merge them. Encoded as a space-separated pair in the `target:` field — dtolnay splits on whitespace.
- Plan B (download page): caught that `'use client'` + `export const metadata` is invalid in App Router. Split into server `page.tsx` (metadata) + client `DownloadView.tsx` (useState + JSX). Also restored the missing `<a` opening tags that the markdown paste had eaten from the spec.
- Tauri bundle targets extended to `["nsis", "msi", "dmg", "deb", "appimage"]` — Tauri picks the right ones per host OS at build time.
- ci.yml + sync-session-memory.yml untouched (verified with `git diff --stat`).
- Landing build green: 33 static pages, 0 errors / warnings. Backend untouched, so the 982-tests baseline is unaffected.

## 2026-05-24 — Batch 3 — Production CI/CD pipeline
- Engineering rule #1: read existing `.github/workflows/ci.yml` fully before touching it. It already covered backend, llm-gateway, security-go, dashboard and Docker images — meaningfully more than the new spec.
- Decision: replace the structure with the new pipeline (three required jobs + `all-green` aggregator) but PRESERVE the unrelated advisory jobs (llm-gateway, security-go, docker). No duplication — the new and old jobs cover disjoint surfaces.
- Verified coverage = 84 % locally before committing → `--cov-fail-under=80` will pass.
- Updated `docker` job's `needs:` to reference the new required-job names (`backend-tests`, `frontend-build`) since the old `backend` / `dashboard` jobs are removed.
- `sync-session-memory.yml` untouched (verified with `git diff --stat`).
- Added a `## Branch Protection (set manually in GitHub)` section to CLAUDE.md under `## Session Management`. The required-check names there match the `name:` strings in the new CI exactly so the rule keys match.
- Full backend suite: 982 passed (unchanged from Batch 2 baseline).

## 2026-05-24 — Batch 2 — Settings + activity log
- Confirmed engineering rule #1 with two greps (`AgentSettings|AgentOverrides|TenantSettings|UserPreference|NotificationSetting` in `database/models.py`; `agent_settings|tenant_settings|user_preferences` in `api/routes/`). Both empty — nothing to duplicate.
- Added three SQLAlchemy models (AgentSettings, TenantSettings, AgentActivityLog) and Alembic revision `d7e2f9b1c3a4`.
- Decided override semantics for AgentSettings: when a row exists it wins; otherwise defaults derive from `Agent.blueprint` so `GET /settings` is never empty. Existing pause/resume endpoints continue to write `Agent.enabled` directly so legacy callers stay green.
- Added single-source helper `agent_runtime/activity_log.py` with idempotent upsert keyed on (agent_id, run_id, event_type). Wired into `OPERARunner._save_run_record` so every OPERA run appends one `run_completed` row.
- New endpoints: `GET/PUT /api/agents/{id}/settings`, `GET /api/agents/{id}/activity`, `GET/PUT /api/tenant/settings`. Validation: `notify_on` and `output_formats` are enum-checked; `threshold_pct` must be in (0, 100]; `updated_by` is required on every agent-settings PUT.
- Asserted in test that PUT /settings does NOT regenerate the blueprint — the signed `Agent.signature` is unchanged after the call.
- Full backend suite: 982 passed (962 baseline + 20 new). No regressions.

## 2026-05-24 — Level 4 chat
- Confirmed backend V1 complete (939 baseline tests; 962 after Batch 1)
- Confirmed Tauri dashboard live (`ABN_1.0.0_x64-setup.exe`, 77.6 MB)
- Confirmed landing 33 pages on Vercel (`/`, 3 layer pages, 5 solution pages, 5 company pages, `/api`, `/transparency`, `/pricing`, `/changelog`, `/status`, `/legal` + 8 legal subpages, 3 auth routes, `/solutions` hub, `/_not-found`)
- Found `frontend/src/pages/AgentDetailPage.tsx` built but not fully hydrated by backend
- Found `Finding` model missing from DB — `AgentRun.output` JSON was the only carrier for findings (no stable id, no per-finding query)
- **Batch 1 started + completed** in same session: Finding model + Alembic migration + persistence helper + 3 new API endpoints + extended `GET /api/agents/{id}` + 23 new tests. All 962 tests green.
- Locked in 8 engineering rules in CLAUDE.md (`## Engineering rules`): grep-first, one source of truth, English-only in code, document the flow, CLAUDE.md is persistent memory, Anthropic-grade safety, idempotency for everything that writes, tests cover the contract.
- Created `JACOB_SESSION.md` + `CHAT_LOG.md` for permanent session memory readable from a fresh chat via the GitHub raw URL.
