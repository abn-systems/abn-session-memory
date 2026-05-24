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
Last updated: 2026-05-24 (after Batch 6)
Repo: https://github.com/abn-systems/ABN
Raw URL (public mirror — auto-synced from main):
https://raw.githubusercontent.com/abn-systems/abn-session-memory/main/JACOB_SESSION.md

## How Jacob uses this file
In any new Claude chat, paste this URL and say "read this":
https://raw.githubusercontent.com/abn-systems/ABN/main/JACOB_SESSION.md

## What is built (as of 2026-05-24)
- Backend: **1010 tests passing** (939 V1 baseline + 23 Batch 1 + 20 Batch 2 + 8 Batch 5 + 20 Batch 6), V1 feature-complete
- Frontend: Tauri desktop dashboard live, Swedish UI, NSIS installer built (`ABN_1.0.0_x64-setup.exe`)
- Landing: 33 prerendered static pages live on Vercel (abn-nine.vercel.app), full dark theme
- Auth: Clerk + RBAC (NODE_ADMIN / AGENT_MANAGER / VIEWER / AUDITOR) + named-employee invite flow live
- AgentDetailPage.tsx: built and **fully hydrated** by the new Finding model + 3 endpoints (Batch 1)
- Post-commit + post-merge git hooks: auto-create Jacob-authored deploy commits so Vercel Hobby plan deploys every push

## Batches done
- Sessions 1-5: Full backend, auth, landing, Tauri dashboard, legal centre, all solution / company pages, mega-menu nav, dark footer, /api, /transparency, /pricing, /changelog, /status — locked in as of session 5
- **Batch 1 (DONE 2026-05-24)** — Finding model + AgentDetailPage hydration
- **Batch 2 (DONE 2026-05-24)** — AgentSettings + TenantSettings + AgentActivityLog
- **Batch 3 (DONE 2026-05-24)** — Production CI/CD pipeline (3 required gates + 80 % coverage)
- **Batch 4 (DONE 2026-05-24)** — Multi-platform Tauri build workflow + download page redesign
- **Batch 5 (DONE 2026-05-24)** — ConnectorsPage live-feed (real-time Observer-Layer events)
- **Batch 6 (DONE 2026-05-24)** — Stripe payments (Subscription + InvoiceLog + billing API + BillingPage)

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

## Batch 2 — Customer settings + per-agent activity log (DONE 2026-05-24)
**Prompt given:** Add AgentSettings, TenantSettings and AgentActivityLog models + Alembic migration; wire AgentActivityLog into OPERA after each run; new endpoints GET/PUT /api/agents/{id}/settings (must NOT regenerate the signed blueprint), GET /api/agents/{id}/activity, GET/PUT /api/tenant/settings; register new router in main.py; 15+ tests; all 962 baseline tests still green.

**Result:**
- Three new tables in `backend/database/models.py`:
  - `agent_settings` — one row per agent, override layer on top of the signed blueprint (schedule, enabled, threshold_pct, output_formats, delivery_emails, notify_on, language, updated_at, updated_by). Override semantics: AgentSettings wins when present, blueprint default otherwise.
  - `tenant_settings` — one row per tenant (language, timezone, currency, date_format, logo_url, primary_color, notification_emails, slack_webhook, updated_at). Separate from `Tenant.policy` (Blueprint Generator governance) and `Tenant.delivery_config` (Type-2 channel routing).
  - `agent_activity_log` — append-only per-agent event stream (run_completed, setting_changed, paused, resumed). Indexed on (agent_id, occurred_at) for fast "last N rows".
- Alembic revision `d7e2f9b1c3a4_add_settings_and_activity_log.py` (depends on `c4f1ad9e8a72`).
- New single source of truth `backend/agent_runtime/activity_log.py::log_run_completed` + `log_setting_changed`. Idempotent per (agent_id, run_id, event_type) — second call updates the row, never duplicates.
- `OPERARunner._save_run_record` now calls `log_run_completed` so every OPERA run automatically appends a "run_completed" activity row alongside the AgentRun + Finding persistence (one place, three side effects, no duplication).
- New endpoints in `backend/api/routes/agents.py`:
  - `GET /api/agents/{id}/settings` — returns AgentSettings row OR blueprint defaults (`is_override: false` when defaulting).
  - `PUT /api/agents/{id}/settings` — upsert, validates enum fields (notify_on, output_formats) + threshold_pct range, stamps `updated_at` + `updated_by`, **does NOT regenerate the blueprint** (signature stays unchanged — asserted in test), writes a `setting_changed` activity row.
  - `GET /api/agents/{id}/activity?limit=N` — newest first, default 30, max 200.
- New file `backend/api/routes/tenant_settings.py` with `GET /api/tenant/settings?tenant_id=…` and `PUT /api/tenant/settings?tenant_id=…`. Registered in `main.py`.
- 20 new tests in `backend/tests/test_settings_and_activity_api.py`. Full backend suite: **982 passed**.

## Batch 3 — Production CI/CD pipeline (DONE 2026-05-24)
**Prompt given:** Replace `.github/workflows/ci.yml` with a production-grade pipeline. Required checks per branch protection: `Backend — 982 tests`, `Frontend — build check`, `Landing — build check`. Backend job runs full pytest with coverage gated at `--cov-fail-under=80`. Target build time under 5 minutes. Do not break `sync-session-memory.yml`. All 982 tests must still pass locally before push.

**Result:**
- Verified local coverage = **84 %** before committing → coverage gate is realistic.
- Rewrote `.github/workflows/ci.yml`:
  - **Three required jobs** (referenced by branch protection):
    - `Backend — 982 tests` — Python 3.12, ruff + mypy (best-effort, `|| true`), pytest with `--cov-fail-under=80`, uploads htmlcov artifact.
    - `Frontend — build check` — Node 20, `npm ci && npm run build` for the Tauri dashboard.
    - `Landing — build check` — Node 20, `npm ci && npm run build` for the Next.js public site.
  - **`all-green` aggregator** depending on the three above — a single status check the branch-protection rule can reference.
  - **Three advisory jobs preserved** from the prior pipeline (no duplication; they cover services not in the new spec): `abn-llm-gateway — test`, `abn-security — vet, build & test`, `Build & push images`. They run alongside but are not required for merge.
- Updated `docker` job's `needs:` array to reference the new job names (`backend-tests`, `frontend-build`) instead of the deleted `backend` / `dashboard` job names.
- `sync-session-memory.yml` untouched.
- Added a `## Branch Protection (set manually in GitHub)` section to CLAUDE.md under `## Session Management`, with the exact required-check names that match the new job names.
- Full backend suite locally: **982 passed**, coverage 84 %.

## Batch 4 — Multi-platform Tauri build + download page redesign (DONE 2026-05-24)
**Prompt given:** Update `frontend/src-tauri/tauri.conf.json` bundle targets from `["nsis"]` → `["nsis", "msi", "dmg", "deb", "appimage"]`. Create `.github/workflows/build-release.yml` with a 7-cell matrix (Windows x64/ARM64, macOS ARM64/x64/Universal, Linux x64/ARM64) using `tauri-apps/tauri-action@v0`, triggered by `release: created` and `workflow_dispatch`. Rewrite `landing/app/company/download/page.tsx` to a three-column macOS/Windows/Linux layout with per-variant download icons + collapsible older-releases accordion. Don't touch `ci.yml`, `sync-session-memory.yml`, backend, or tests.

**Result:**
- `tauri.conf.json` bundle targets extended to 5 formats; Tauri auto-selects per OS at build time.
- New workflow `.github/workflows/build-release.yml`:
  - 7-cell matrix across `windows-latest`, `macos-latest`, `ubuntu-22.04`.
  - **Bug fix vs the spec:** the spec passed `--target X` (a flag string) into `dtolnay/rust-toolchain`'s `targets:` input — which expects raw target triples. Split the matrix into `target` (for rust-toolchain) and `args` (for tauri-action) so every cell actually runs.
  - macOS Universal cell installs BOTH `x86_64-apple-darwin` and `aarch64-apple-darwin` so Tauri's `--target universal-apple-darwin` lipo-merge works.
  - All Apple signing + Tauri-updater secrets wired via env vars; absent secrets ⇒ unsigned build (Tauri behaviour).
  - `ci.yml` and `sync-session-memory.yml` left untouched (verified with `git diff --stat`).
- Download page rewritten:
  - **Split into two files** (server `page.tsx` + client `DownloadView.tsx`) so `export const metadata` (server-only) and `'use client'` (needed for `useState`) can coexist. The spec collapsed both into one file which Next.js App Router rejects.
  - Three-column grid per release block: macOS (3 variants), Windows (4), Linux (6).
  - Each variant: live download icon (only when `coming: false` AND `file: not null`) → `${RELEASE_BASE}/download/v${version}/${file}`, else "Snart" placeholder.
  - "View release notes →" per release (one English phrase; everything else Swedish).
  - Older releases collapsed inside `<button>`-driven `useState`-tracked accordions; `OLDER_RELEASES: Release[] = []` ready for v1.1+ entries.
  - **Restored the `<a` opening tags** that got eaten in the spec's markdown paste — otherwise the file wouldn't parse.
- `cd landing && npm run build` → ✓ 33 static pages, zero errors / warnings.

## Batch 5 — ConnectorsPage live-feed (DONE 2026-05-24)
**Prompt given:** Add a Live-flöde panel to ConnectorsPage that polls the last 10 Observer-Layer events every 30 s. Build only what's missing — don't rewrite ConnectorsPage.tsx or MarketplacePage.tsx. New backend endpoint `GET /api/connectors/live-feed?tenant_id=&limit=`. New frontend helpers `getLiveFeed` (client.ts) and `formatRelative` (format.ts). 5+ tests. All 982 baseline tests still green.

**Result:**
- Backend (`backend/api/routes/connectors.py`): new `GET /api/connectors/live-feed` declared BEFORE the int-typed `GET /{connector_id}` so the literal-path match wins (regression test included). Returns `LiveFeedEvent[]` — `id, domain, event_type, case_key, source_system, created_at`. Tenant-scoped; `limit` capped at 50 server-side. Reads existing `Event` rows that already passed Trust-Layer scrubbing — no further redaction needed.
- Frontend `client.ts`: appended `LiveFeedEvent` interface + `getLiveFeed(tenantId, limit=10)` using the existing axios `api` wrapper (not raw `fetch` — matches the other 20+ helpers in the file). Failure semantics: returns `[]` instead of throwing so a Node with zero events shows an empty panel, not a red error banner.
- Frontend `lib/format.ts`: added `formatRelative(iso)` — relative time ("Xs sedan", "X min sedan", "X tim sedan") with absolute-date fallback for >24 h. Pure, dependency-free.
- Frontend `pages/ConnectorsPage.tsx`: extended imports + added `<LiveFeed tenantId={tenantId} />` invocation directly before the closing `</div>` of the existing page body. New `LiveFeed` component is a sibling function in the same file — the existing `ConnectorsPage` function body is unchanged (rule "do not rewrite"). Renders domain-colour-coded chips (Bokföring/Schema/Logistik/Dokument), `event_type · case_key`, relative timestamp + the No-Data footer line.
- 8 new backend tests in `tests/test_live_feed_api.py`: empty list, newest-first ordering, default-limit 10, explicit-limit, 50-row cap, strict tenant scoping, literal-path-vs-int-param regression check, response shape.
- Full backend suite: **990 passed** (982 + 8). Frontend: typecheck clean, **60 tests green**, Vite build green.

## Batch 6 — Stripe payments (DONE 2026-05-24)
**Prompt given:** Add Stripe billing — `Subscription` + `InvoiceLog` models, Alembic migration, billing API (`/checkout-session`, `/portal-session`, `/subscription`, `/webhook`), BillingPage on the dashboard, wire route + nav, `stripe==8.7.0` in requirements, 15+ tests. Webhook signature ALWAYS verified (fail-closed). All 990 baseline tests still green.

**Result:**
- Five spec bugs fixed during the build (Engineering rule #6, fail-closed > spec-literal):
  1. `from core.database import get_db` → `from database.session import get_db` (the spec path doesn't exist).
  2. `from core.logging import get_logger` → stdlib `logging.getLogger` (no such module).
  3. `STRIPE_*` UPPER_CASE field names → lowercase to match existing Settings convention (`case_sensitive=False` handles UPPER_CASE env vars on the `.env` side).
  4. `stripe.error.SignatureVerificationError` → `stripe.SignatureVerificationError` (the alias namespace was removed in stripe-python 8.x).
  5. `invoice.paid` tenant resolution stored the Stripe customer-id in the `tenant_id` column. Fixed: now looks up the Subscription by `stripe_customer_id` to map back to ABN's tenant_id.
- New models: `Subscription` (one row per tenant — plan, status, stripe ids, period-end, cancel flag) and `InvoiceLog` (append-only receipts, idempotent on `stripe_invoice_id`). Both in `backend/database/models.py`. No card data, no bank data, no PII — Stripe holds the payment instrument.
- Alembic revision `e5b8c1f3d29a_add_subscriptions_and_invoice_log.py` (depends on `d7e2f9b1c3a4`).
- New `backend/api/routes/billing.py`:
  - `POST /api/billing/checkout-session` — creates Stripe Checkout, returns `checkout_url`. 503 when Stripe unconfigured (fail-closed safety on fresh installs).
  - `POST /api/billing/portal-session` — Stripe Customer Portal; 404 when no Subscription.
  - `GET  /api/billing/subscription` — returns row state, or starter/inactive defaults for new tenants.
  - `POST /api/billing/webhook` — signature verification FIRST (fail-closed); handles `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`. Idempotent across all four. Unknown events return 200 so Stripe doesn't retry indefinitely.
- `core/config.py` extended with `stripe_secret_key`, `stripe_webhook_secret`, `stripe_professional_price_id`, `stripe_starter_price_id`. All empty by default → billing routes 503 until real values supplied.
- `backend/.env.example` extended with the four UPPER_CASE env vars.
- `backend/requirements.txt` extended with `stripe==8.7.0`.
- `main.py` registers the new router alongside the others.
- New frontend `BillingPage.tsx` on the `/billing` route — two plan cards (Starter free / Professional €299/mo), "Uppgradera till Professional" → Stripe Checkout redirect, "Hantera prenumeration" → Stripe Customer Portal redirect, period-end notice + cancel-at-period-end orange flag.
- `frontend/src/api/client.ts` extended with `SubscriptionStatus` interface + `getSubscription` / `createCheckoutSession` / `createPortalSession` helpers (paths WITHOUT `/api` prefix — the axios baseURL already includes it; spec had double prefix bug).
- `App.tsx` routes `/billing` → `<BillingPage />` (additive).
- `Layout.tsx` nav extended with "Fakturering" entry + new IconBilling SVG (additive).
- 20 new tests in `backend/tests/test_billing_api.py` — 503 fail-closed gates, checkout-session happy path + 422 unknown plan + 404 unknown tenant + starter-price-id selection, portal-session 404 + happy path, webhook signature gate (missing header / invalid sig / unconfigured secret), all four event handlers, idempotency on replay, unknown-event-type 200, tenant lookup via stripe_customer_id (not from payload).
- Full backend suite: **1010 passed** (990 + 20). Frontend: typecheck clean, **60 tests green**, Vite build green. Every Stripe SDK call mocked — zero real network calls in tests.

## Batch 7 — NEXT
www.abnplatform.com DNS + real public API endpoint.

## Batch 8
Fortnox end-to-end (needs org number + API key from Jacob).

## Batch 9
Branch-protection hardening (manual GitHub step — listed in CLAUDE.md under `## Branch Protection`).

## Manual prereqs for Stripe to go live
1. Stripe Dashboard → create two Products + Price objects (Starter free / Professional €299/mo recurring) and copy their `price_...` ids.
2. Stripe Dashboard → Developers → API keys → copy the `sk_test_...` (or `sk_live_...`) secret key.
3. Stripe Dashboard → Developers → Webhooks → add endpoint `https://api.abnplatform.com/api/billing/webhook`, subscribe to `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`. Copy the `whsec_...` signing secret.
4. Set all four env vars on the Node (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PROFESSIONAL_PRICE_ID`, `STRIPE_STARTER_PRICE_ID`). The billing routes return 503 until they are set.

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
