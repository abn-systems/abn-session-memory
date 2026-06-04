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

## ÅTERHÄMTNING EFTER /clear
1. cd C:\Users\Jacob\Downloads\abn
2. git log --oneline -5
3. git status
4. cat CLAUDE.md | head -40
5. cat JACOB_SESSION.md

ABN lever i: GitHub + denna disk + dessa markdown-filer. Aldrig i chatt-minnet.

## JUST NU
Status: Batch 76a0i (MND) pågår — real finding-production: shared-context threading + single accumulation + PERMANENT sim-guard (Option A+B); flag-gated. Branch feat/batch-76a0i-mnd-finding-production.
Repo-sökväg: C:\Users\Jacob\Downloads\abn
Main-branch: 03d9b7e — charter #92 + 73/74/75 på main. 76a0i (this) på sin branch tills PR mergas.
Senast: 2 read-only Discoveries → live OPERA producerar 0 riktiga findings (fresh-context-per-step isolation) OCH read_* simar ovillkorligt (inga adapters wired). Jacob valde split: 76a0i (P1+P2+P3+flaggor) nu; P4 (PII-on-findings) → 76a.
Nästa: 76a0i = tråda EN delad context (read→analyze→report samarbetar) + läs findings EN gång (ingen dubbelräkning) + ingen adapter→genuint tomt (allow_simulated_findings default-False) + needs_connector-reason + 2 flaggor (threading default-ON). Sedan 76a (RAL finding-shape, nu meningsfullt) → 76b → P2a → P2b.
VIKTIGT: PERMANENT regel — ingen fabricerad/simulerad finding får nå en prod/kund-yta. threading default-ON (prod-säkert: tomt tills riktig adapter wired). P4 EXKLUDERAD. Skydda 73/74.

## ABN ROADMAP v2 (authoritative — Jacob's nano-detail canon)
THREE GUIDING STARS (apply to every track): (1) **3 master gates** —
GraphAdmissibility, AgentNeed, BlueprintReadiness — ALL other scores feed
these three (anti score-chaos); (2) **hard floors > blended scores** (a junk
sub-score must veto, not average away); (3) **unbroken chain of proof**:
DNA→Graph→Genesis→Compiler→Simulation→Runtime→Verification→Audit→Tuari.
Build weights as MECHANISM, calibrate against real data (no false precision).
MND batches build nano-detail from canon; metaphors (Google-Translate =
ABN-WIR; coffee-machine = Autonomous-Engine; chip/robot-kit =
Component-Manifest) are quoted into the relevant track's prompt.

- **Spår 1 — motor-integritet:** [73 ✅ run() status honesty] [74 ✅ Critic+ESE
  on the live path] → **75 (this) Failure Taxonomy v1** → Build 2 (RAL
  attestation + cross-reference + Finding.attested write-back) → P2(a) hard
  conformance floor + VariantConformanceMap → P2(b) 5-level SimulationGate /
  SafeFailureRate.
- **Spår 2 — Safety Spine** (reads the Failure Taxonomy): PhaseContractValidator,
  RunStatusReducer (weakest-link), RuntimeStateMachine + TransitionGuard,
  RunLock, idempotency-key, Transactional Outbox, BlueprintQualityScore
  hard-floors, Signed BrainProfile / ToolBinding, ABN-RCU snapshots,
  AgentHealth (consumes auto_pause_weight), Safe Mode, Supervisor
  Reconciliation.
- **Spår 3 — DNA / Agent-OS:** A Observer Mode 0–6 / Event-Envelope / case-key /
  no-loss / Permission-Manifest; B ABN-WIR (translation core) / 6 graph levels /
  GraphAdmissibility / Work-Units / GenesisPackage; C Autonomous-Engine-as-
  compiler / BrainProfile + BrainRiskFit / ToolFit-floors / Verification-
  Compiler; D Trust-Kernel / Coordinator / Jurisdiction / File-Engine; E local
  Control-Plane SQLite / AgentHealth; F Evolution / Shadow / Kill-switch /
  Abstract-Feedback; G Tuari control room.
- **Spår 4:** internal brains (17-step loop), agents-in-daily-tools
  (Gmail/Teams/Slack), internal chat-panel, V1 launch → reshaped vision.

## TODO — Design-inspiration från Claude desktop-appen
Jacob: "Ta inspiration från Claude-appen — de har chat, kod, design, fungerar utan problem. ABN-appen ska vara så snabb och bra." Den nya v7 sage-designen är på plats men UX-flowet (chat-tab på AgentDetailPage, kod-blocken på /api, navigationen i sidofältet) kan slipas mot Claude-appens kvalitet i en framtida batch. Notering för senare — inte i scope för Batch 35.

Last updated: 2026-05-28 (after Batch 34)
Repo: https://github.com/abn-systems/ABN
Raw URL (public mirror — auto-synced from main):
https://raw.githubusercontent.com/abn-systems/abn-session-memory/main/JACOB_SESSION.md

## How Jacob uses this file
In any new Claude chat, paste this URL and say "read this":
https://raw.githubusercontent.com/abn-systems/abn-session-memory/main/JACOB_SESSION.md

## What is built (as of 2026-05-25)
- Backend: **1193 tests passing** + abn-security Go suite all-green (V1 baseline 939, then per-batch additions through Batch 22B; full per-batch accounting in the dated entries below), V1 feature-complete; TIER 3 EXECUTE_CHANGE now end-to-end functional (opt-in)
- **Production deploy live** (Batch 7): Hetzner CAX21 server `abn-core` at 46.225.165.62 (Frankfurt, Ubuntu 24.04, GDPR DPA signed). All four containers running under Docker Compose at `/opt/abn`. `https://api.abnplatform.com/health` returns `{"status":"ok","env":"production"}`. Vercel landing on `abnplatform.com` + `www.abnplatform.com` both Valid Configuration.
- Frontend: Tauri desktop dashboard live, Swedish UI, NSIS installer built (`ABN_1.0.0_x64-setup.exe`)
- Landing: 33 prerendered static pages live on Vercel (abn-nine.vercel.app), full dark theme
- Auth: Clerk + RBAC (NODE_ADMIN / AGENT_MANAGER / VIEWER / AUDITOR) + named-employee invite flow live
- AgentDetailPage.tsx: built and **fully hydrated** by the new Finding model + 3 endpoints (Batch 1)
- MarketplacePage.tsx: **wired to real DB** (Batch 7) — install / disable / re-enable via the existing `Connector` table; no fake client-side Set anymore
- Post-commit + post-merge git hooks: auto-create Jacob-authored deploy commits so Vercel Hobby plan deploys every push

## Batches done
- Sessions 1-5: Full backend, auth, landing, Tauri dashboard, legal centre, all solution / company pages, mega-menu nav, dark footer, /api, /transparency, /pricing, /changelog, /status — locked in as of session 5
- **Batch 1 (DONE 2026-05-24)** — Finding model + AgentDetailPage hydration
- **Batch 2 (DONE 2026-05-24)** — AgentSettings + TenantSettings + AgentActivityLog
- **Batch 3 (DONE 2026-05-24)** — Production CI/CD pipeline (3 required gates + 80 % coverage)
- **Batch 4 (DONE 2026-05-24)** — Multi-platform Tauri build workflow + download page redesign
- **Batch 5 (DONE 2026-05-24)** — ConnectorsPage live-feed (real-time Observer-Layer events)
- **Batch 6 (DONE 2026-05-24)** — Stripe payments (Subscription + InvoiceLog + billing API + BillingPage)
- **Batch 7 (DONE 2026-05-25)** — MarketplacePage wired to real backend (`initiateConnect` / `disableConnector` / `enableConnector`, real installed state from `Connector` table, OAuth redirect support). 8 new backend tests. 1018 total.
- **Batch 8 (DONE 2026-05-25)** — Migration deploy fix: replaced `Base.metadata.create_all()` startup with `alembic upgrade head` in `main.py` lifespan. No new tests (0 net — startup-path change only); 1018 still pass at 84.35% coverage. Manual one-time `alembic stamp head` required on the Hetzner production DB before the first restart — documented in CLAUDE.md `## Production DB — one-time stamp`.
- **Batch 9 (DONE 2026-05-25)** — Full automation chain wired: connector activate → DNA → Process Graph → Agent creation runs as a background task on every `POST /api/connectors/{type}/connect`. New `AgentScheduler` (daemon thread, 60 s tick, blueprint-cron driven, per-tenant lock). Plan limits enforced (starter=2 / professional=25 / enterprise=∞) via `enforce_agent_limit_or_raise`. Observer `CONNECTOR_RESOURCES` now derived from the YAML catalogue — all 18 catalogue types covered (delivery-only YAMLs yield empty resource lists). 18 new tests. Backend: **1036 passed**, 84.45% coverage.
- **Batch 10 (DONE 2026-05-25)** — DNS + public API endpoint wired: `landing/vercel.json` carries the apex→`www` 308 redirect; `landing/next.config.mjs` adds five global security headers (HSTS / X-Frame-Options / X-Content-Type-Options / Referrer-Policy / Permissions-Policy); `Settings.public_api_url` + surfaced on `GET /health` (`"public_url"` field); CORS allow-list extended with `https://www.abnplatform.com` + `https://abnplatform.com`. CLAUDE.md gains a `## DNS & Domains` section with the manual registrar + Vercel steps Jacob still has to perform. 1036 tests still green, 33 landing pages still build clean.
- **Infra event (2026-05-25)** — Production deploy: domain DNS + email + Hetzner host + Docker Compose + Nginx/TLS + live `api.abnplatform.com`. Tracked here for cross-session memory; not numbered as a feature batch.
- **Batch 22B (DONE 2026-05-26)** — TIER 3 EXECUTE_CHANGE second half: EXECUTOR archetype + AVM tier-3 rules + Fortnox write-back + RollbackRecord + OPERA pre-flight + rollback API. New `Tenant.tier3_enabled` column (default False — every tenant opted-out until DPA addendum signed) + `RollbackRecord` table, both in Alembic `b32c7e2ba5e5` (raw SQL `ALTER TABLE / CREATE INDEX IF NOT EXISTS` per Batch 12/16/17/18 lesson). `_derive_archetype_handbook` gained `tier3_enabled: bool = False` kwarg — returns EXECUTOR only when tenant opted-in + platform `enable_execute_change=True` + blueprint has an EXECUTE_CHANGE step. AVM step 5b downgrades (never rejects) EXECUTOR → PLANNER + tier 3 → 2 when: regulated industry forbids EXECUTE_CHANGE OR EXECUTOR without `execute_change=true` flag OR tier 3 unsigned OR EXECUTE_CHANGE step lacks `rollback_action`. Three Fortnox write capabilities (`fortnox_update_invoice_status` — `cancelled`/`reminded` only, never `paid`; `fortnox_create_reminder`; `fortnox_update_cost_center`) each do three-layer opt-in check (platform + per-system + per-tenant) → verify_blueprint_via_security_service (cross-process to abn-security) → GET prior state → PUT new state → persist RollbackRecord. New `KIND_EXECUTE` constant + 5th `execute` category in ACTION_MAP. OPERA `_tier3_preflight_or_raise` runs before Observe phase — tier 1/2 skip cost-free, tier 3 verifies signature via abn-security or raises `TierThreeVerificationError` (new `agent_engine/errors.py`). New API: `GET /api/admin/rollback-records` + `POST /api/admin/rollback/{id}` (NODE_ADMIN only, idempotent, audit-logged via ABNActivityLog). `_ROLLBACK_DISPATCH` map routes action_type → reverse Nango call. New env vars `ENABLE_EXECUTE_CHANGE` + `FORTNOX_WRITE_ENABLED` (canonical names; both default False). 13 new tests in `test_tier3_executor.py` + 1 test renamed (`test_action_map_has_all_four_categories` → `all_five_categories`). Backend: **1193 passed** (1180 + 13). Go: 7/7 packages green. Landing: 33/33. **Architectural decisions:** (a) Two tier-3 switches (platform `enable_execute_change` + per-tenant `tier3_enabled`) — DPA addendum is platform-level legal opt-in; tenant flag is operational opt-in; both required (defence-in-depth); (b) AVM ALWAYS downgrades, never rejects — keeps human-in-the-loop pattern active even on misconfigured blueprints (rule #6); (c) `paid` status deliberately NOT in allowlist — financial-record changes need auditor sign-off, V1 stays at the safe boundary; (d) signature verified cross-process TWICE per tier-3 step (Mission Layer pre-flight + per-capability re-verify) — Mission Layer bug can't bypass the gate. **What's next (Batch 22C):** per-tenant DPA addendum PDF, `Tenant.tier3_dpa_signed_at` column, "Aktivera Tier 3" panel on AdminPage with consent capture, store `connection_id` on RollbackRecord at write-time.
- **Batch 22A (DONE 2026-05-26)** — TIER 3 foundation: abn-security compiled + cross-language blueprint signatures. The Go service that's been in the repo unchecked since handbook ch. 11 was first scaffolded is now **compiled, vetted, and tested for the first time** — under `golang:1.22-alpine` via local Docker, matching what the CI `security-go` job has always run. New Go package `abnsecurity/internal/blueprint` (stdlib-only, 75% coverage) with `BlueprintVerifier` (HMAC-SHA256 over canonical JSON + tier-3 flag enforcement + audit logging via the existing `audit.Logger`). New HTTP endpoint `POST /verify-blueprint` on abn-security; request `{"blueprint": "<base64 JSON>"}`, response `VerifyResult{valid, reason, agent_id, tenant_id, tier}`, statuses 200/400/503 (503 when `ABN_NODE_SIGNING_KEY` empty — fail-closed). New Python module `backend/trust/blueprint_signer.py` — single source of truth for the canonical byte stream (`json.dumps(bp, sort_keys=True, separators=(",", ":"))`). The legacy `blueprint_generator.sign_blueprint` and `verify_blueprint` are now thin wrappers that delegate to it — existing import paths backward-compatible, but the canonical form (and the default key, now `node_signing_key`) is unified. New `verify_blueprint_via_security_service` helper for the cross-process pre-flight (fail-closed on any error). Settings + `.env.example` get `NODE_SIGNING_KEY` and `ABN_SECURITY_SERVICE_URL` entries (the legacy `BLUEPRINT_SIGNING_KEY` stays for backward compat). Generator emits an explicit warning when `node_signing_key` is unset or set to the placeholder default. Closes the **abn-security-not-compiled known-issue** from CLAUDE.md (open since handbook ch. 11). 9 new Go tests + 9 new Python tests (incl. an explicit Go-↔-Python interop assertion: `test_canonical_form_matches_go_reference` computes the expected signature byte-by-byte and compares — drift detector). Backend: **1180 passed** (1171 + 9). Go: 7/7 packages pass. Landing 33/33. **Architect deviations from spec, all rule-driven:** (a) New package `abnsecurity/internal/blueprint` (module is `abnsecurity`, not the spec's `github.com/abn-systems/abn-security`); (b) Reused `ABN_NODE_SIGNING_KEY` env var instead of spec's new `ABN_SIGNING_KEY` (rule #2 — one env var per secret); (c) Canonical signer module is the new SSoT — legacy `sign_blueprint`/`verify_blueprint` migrated to delegate (rule #2); (d) `WithBlueprintVerifier` builder method on `api.Server` so the constructor signature stays backward-compatible. **What's next (Batch 22B):** `EXECUTOR` archetype, AVM tier-3 rules, `Fortnox.create_invoice` write capability, OPERA Mission Layer pre-flight verify, write-back rollback semantics.
- **Batch 21 (DONE 2026-05-26)** — Wire Proposal creation into OPERA runtime. New `backend/agent_runtime/proposals_persistence.py` with `save_proposal_from_run(db, *, agent_id, tenant_id, run_id, blueprint, output, impact_eur, confidence) -> Optional[int]`. Mirrors `findings_persistence.save_findings_from_output` shape exactly (rule #2): kwargs-only, idempotent per `(run_id, tenant_id)`, fail-silent at every step. The helper calls `evaluate_commitment` internally (avoids threading `commitment_decision` through `OPERARunResult` — keeps the runner unaware of the gate). Only writes when the gate's decision is `save_proposal` (tier 2 + confidence ≥ 0.50); tier-1 + below-floor are no-ops. Derives `action_type` from `blueprint.steps[]` via `_PROPOSE_ACTION_MAP` (`propose.schedule_change → "schedule_change"`, `propose.cost_reduction → "threshold_adjustment"`, `propose.draft_email → "report_delivery"`, etc); unknown propose-class actions default to `"agent_pause"` (safest fallback — proposal still surfaces, operator pauses agent for review). Derives `target_system` from `blueprint.delivery` / `connectors` / `active_connections` in that fallback order. `impact_summary` is aggregate-only ("3 fynd · potentiell besparing EUR 18 600"). Wired in `OPERARunner._save_run_record` between the Agent Memory writes (Batch 17) and the Batch-18 notification dispatch block — the existing Batch-18 query-and-fan picks up the newly-created rows automatically; **deliberately did NOT add a redundant `deliver_proposal_notification` call** (rule #1, the spec's TASK 2 second half would have duplicated Batch 18). **Deliberately skipped TASK 3** (draft_email wiring from commitment_gate) — the gate doesn't emit `draft_email`; Batch 20's capability is invoked directly via the `propose.draft_email` ACTION_MAP entry, separate concern. **Deliberately skipped TASK 4 OPERARunResult mutation** — helper computes the decision internally; threading a dataclass field gains nothing. 10 new tests in `tests/test_proposal_wiring.py` covering helper correctness (tier-2 creates row, agent_id/tenant_id/status/impact correct), resilience (fail-silent on broken DB, tier-1 no-op, idempotent return), and end-to-end (Batch-18 query finds Batch-21 row, `GET /api/proposals` returns the new row with `status="pending"`). Resolves the Batch-18 "no production OPERA path creates Proposal rows" gap flagged in CLAUDE.md. Backend: **1171 passed** (1161 + 10), no regressions. Landing 33/33. Frontend typecheck clean (no frontend changes).
- **Batch 20 (DONE 2026-05-26)** — Email capability (read inbox → draft → human approves → never auto-send). New `EmailDraft` model + Alembic revision `05da8d0f9d26` with `to_fingerprints` / `cc_fingerprints` (SHA-256 lists, never raw addresses) + `subject_template` / `body_template` / `context_summary` (templates validated by `_validate_template_no_raw_values` — rejects 4+ digit runs and literal `@domain` strings). Extended `gmail.yaml` + `outlook.yaml` with `direction: both` + `resources[]` declaring the messages endpoint, 5-min poll, full block list for subject/body/raw addresses. New normalisers `observer/normalisers/gmail_messages.py` + `outlook_messages.py` — only `id` / `thread_id` / `event_time` / `label_ids` / `has_attachments` / `from_domain` survive; PII Guardian runs as defensive second pass. New `backend/agent_engine/capabilities/email.py` with three functions: `read_inbox` (filters pre-normalised events by from_domain / labels / attachments), `draft_email` (persists pending EmailDraft, fingerprints recipients, validates templates), `send_approved_draft` (refuses to run unless `status == "approved"` — defence-in-depth on top of API RBAC). Registered in `CAPABILITY_REGISTRY` + ACTION_MAP gets `fetch_data.gmail`, `fetch_data.outlook`, `propose.draft_email`, `report.send_approved_email`. New `backend/api/routes/email_drafts.py` — `GET /api/agents/{id}/drafts` + `GET /api/drafts` + `POST /api/drafts/{id}/approve` + `POST /api/drafts/{id}/reject`. NODE_ADMIN | AGENT_MANAGER. Approve handler cross-checks supplied cleartext recipients against stored fingerprints; mismatch refuses send. Writes `ApprovalRecord(proposal_id="draft:{id}")` — reuses the existing Proposal audit table (rule #2). API never returns `to_fingerprints` / `cc_fingerprints` — only `recipient_count` / `cc_count`. New frontend `DraftsPage.tsx` at `/drafts` route (sidebar "Utkast"), two sections (pending with ✅/❌ + approve/reject modals, resolved greyed-out), 30s TanStack Query refetch, Swedish copy. 13 new tests covering normaliser blocking + domain-only extraction (4), draft creation idempotency + fingerprinting + raw-value rejection (3), send guard refuses non-approved (2), API privacy contract + decisions + ApprovalRecord writes (4). Fixed one pre-existing test assertion in `test_delivery.py` that asserted `gmail.direction == "outbound"` (now `both` since Batch 20 extended it). Backend: **1161 passed** (1148 + 13), frontend 60 tests + Vite build OK, landing 33/33.
- **Batch 19 (DONE 2026-05-26)** — Admin Dashboard + ABN Node landing copy. New `backend/api/routes/admin.py` with three endpoints — `GET /api/admin/overview` (aggregate dashboard: tenant + plan, agent counts, 7-day runs/findings/impact_eur from ROILedger, pending proposals, users + node-admin cap, connector health, scheduler statuses, DB + LLM-gateway health, subscription), `GET /api/admin/activity-log` (filterable abn_activity_log with activity_type + days + limit query params), `GET /api/admin/schedulers` (live status() from all three scheduler singletons). All three guard via the existing `require_role(Role.NODE_ADMIN)` dependency. **Surface A only — never cross-tenant.** Surface B (fleet admin) audited + rejected: would invert the No-Data Guarantee; Stripe Dashboard + GitHub Releases + external uptime cover ABN-internal needs without that. New `frontend/src/pages/AdminPage.tsx` at `/admin` route, four sections (Overview tiles, Systemstatus, Användare, Aktivitetslogg) mirroring BillingPage.tsx pattern (TanStack Query 30s refetch, same Card/StatusBadge/Table primitives, Swedish copy). New "Admin" sidebar link with shield icon. Landing copy: `"Ladda ner ABN"` → `"Ladda ner ABN Node"` headline + page metadata; new "Vad är ABN Node?" explainer block above the download accordion (two short Swedish paragraphs about per-customer isolated Nodes). 9 new tests in `tests/test_admin_routes.py` covering auth (4: requires_node_admin × 2, non_admin_gets_403, unauthenticated), shape (1: full payload contract), correctness (3: agent counts, 7-day window filtering, proposal pending), schedulers (1). Reuses `require_role` / `MAX_NODE_ADMINS_PER_ORG` from auth.rbac (rule #1 — extend, never duplicate the RBAC machinery). Backend: **1148 passed** (1139 + 9), frontend: 60 tests + Vite build OK, landing: 33/33 static pages.
- **Batch 18 (DONE 2026-05-26)** — Teams/Slack notifications + approval buttons. Extended `delivery/slack_deliverer.py` with `deliver_proposal` (Block Kit `blocks[]` with header + fields + actions block; two `button` elements styled primary/danger; `value` carries `"<approve|reject>:<proposal_id>:<dispatch_token>"`). Extended `delivery/teams_deliverer.py` with `deliver_proposal` (Adaptive Card v1.4 with TextBlock + FactSet + two `Action.Submit` actions). New `delivery.router.compute_dispatch_token` (HMAC-SHA256) + `verify_dispatch_token` (constant-time) + `DeliveryRouter.deliver_proposal_notification` (Slack first, Teams fallback, refuses to dispatch without `NOTIFICATION_SIGNATURE_SECRET` — fail-closed). New `NotificationDispatch` table (Alembic `4b0b88e1b9f7`) correlates `message_id → proposal_id` with status lifecycle. Two new routes: `POST /api/slack/interactions` (HMAC `v0:{ts}:{raw}` signature verify against `SLACK_SIGNING_SECRET`, 5-min replay window, always 200 OK) + `POST /api/teams/interactions` (V1 bearer-token check against `TEAMS_VALIDATION_TOKEN`, JWT-via-JWKS as documented V2 path). Both call the existing `proposals._record_decision` (rule #2 — single source of truth for the approval contract — never re-implemented). OPERA `_save_run_record` fires `deliver_proposal_notification` for any pending Proposal rows created during the run (fail-silent). Three new Settings: `slack_signing_secret`, `teams_validation_token`, `notification_signature_secret` — all default empty so a fresh install never accepts unverified interactions. 14 new tests in `tests/test_slack_teams_notifications.py` covering Block Kit / Adaptive Card structure, HMAC token, dispatch persistence, signature verify, replay-attack rejection, approve+reject flows, idempotent double-approve, always-200-OK, and the Teams approve flow. Reuses Stripe webhook pattern (signature-verify-then-act + always-200), existing `BaseDeliverer._proxy` (Nango plumbing), `ApprovalRecord` model (immutable audit). Backend: **1139 passed** (1125 + 14), landing 33/33.
- **Batch 17 (DONE 2026-05-26)** — Agent Memory (three layers). New `backend/agent_memory/` package — `short_term.py` (7-day TTL, reinforcement counter, no_raw_values_check policy), `long_term.py` (promotion at >=3 reinforcements, decay 0.8 at 6 mo inactive, never deletes). Extended `agent_evolution/evolution.py` with `persist_state` + `load_state` (TTL-cached read-through). Three new tables (`agent_evolution_state`, `agent_short_term_memory`, `agent_long_term_memory`) in one Alembic revision `7ab8394c998a` — raw SQL `CREATE INDEX IF NOT EXISTS` per Batch 16 lesson. OPERA Plan phase now loads memory via `_load_agent_memory` (fail-silent, safe-empty defaults). `_save_run_record` writes short-term for findings with conf ≥ 0.7, persists evolution state, runs long-term promotion — all fail-silent. New API routes `GET /api/agents/{id}/memory` + `GET /api/agents/{id}/evolution` — aggregate metadata only, `fact_key` and `fact_payload` never exposed via API even though they're No-Data-safe (stable correlation handle = metadata leakage). 14 new tests in `tests/test_agent_memory.py`. Reuses `trust/fingerprint.py` (memory keys), existing `AgentEvolutionEngine.compute_state` (unchanged, persistence added in place), `findings` table (source of reinforcements). Backend: **1125 passed** (1111 + 14), landing 33/33.
- **Batch 16 (DONE 2026-05-26)** — Friday Report viral-loop e-mail. New `backend/friday_report/` package (generator, email_builder, sender, scheduler). Aggregates ROILedger + AgentRun + Agent rows for the closed Sat→Fri week into `FridayReportData` (EUR saved, hours saved, agents active, runs completed, top-5 agents). HTML+plaintext e-mail with ABN purple `#5A3FC0` header band + gold `#F2C94C` money accent, Swedish copy, deterministic render. SendGrid sender with API-key auth (skips Nango — Friday Report aggregates ARE the No-Data-safe figures, so SendGridDeliverer's number-strip would be wrong here). Scheduler is a daemon thread mirroring `AgentScheduler` — cron `"0 7 * * 5"` Europe/Stockholm with ±5 min grace window, three-layer idempotency (cron + already_sent_this_week guard + `(tenant_id, week_ending)` unique index). New `FridayReportLog` table — Alembic revision `e0dde7bb974c`. API: `GET /api/friday-report/preview` (read-only, accepts optional `report_date` query), `POST /api/friday-report/send` (manual trigger, dry_run support), `GET /api/friday-report/history`. Wired into `main.py` lifespan after `AgentScheduler`. 12 new tests in `tests/test_friday_report.py` covering aggregation correctness, deterministic e-mail render, idempotent sends, dry-run, failure capture, scheduler already-sent guard, and the 3 API routes. **Bonus hotfix:** discovered + fixed the `8f8acda9625d_align_schema_drift` migration — `postgresql_if_not_exists=True` is NOT honoured on SQLAlchemy 2.0 `Index()` objects (only on `Table.create()`), so the prior Batch 12 migration would have crashed on Hetzner. Rewrote all 7 `op.create_index(..., postgresql_if_not_exists=True)` calls to raw SQL `op.execute("CREATE INDEX IF NOT EXISTS ...")`. Backend: **1111 passed** (1099 + 12).

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

## Production deploy event — DNS + email + Hetzner + Docker + TLS (2026-05-25)
(Originally logged as Batch 7; relabeled to an infra event so the Batch 7 slot can hold the MarketplacePage wire-up per the user's explicit numbering. Facts below intact.)
**Goal:** flip `abnplatform.com` from a marketing-only site into a real production stack — domain email, public landing, live `api.abnplatform.com` host serving the FastAPI backend.

**Delivered:**
- **Proton Mail domain** `abnplatform.com` fully configured — VERIFIED, MX, SPF, DKIM, DMARC all green. Three mailboxes live:
  - `hello@abnplatform.com` — sales / general
  - `reports@abnplatform.com` — automated agent-report sender
  - `legal@abnplatform.com` — legal / DPA / security
- **Vercel DNS** fixed — both `abnplatform.com` and `www.abnplatform.com` show **Valid Configuration**. Landing site continues to deploy from `origin/main` via the Jacob-authored post-merge hook.
- **Hetzner CAX21 server** provisioned — name `abn-core`, IP `46.225.165.62`, Frankfurt region, Ubuntu 24.04 LTS, GDPR-compliant DPA signed with Hetzner.
- **Docker + Docker Compose** installed; repo cloned to `/opt/abn`. All four containers running:
  - `abn-postgres` (PostgreSQL 16, persistent volume)
  - `abn-core` (FastAPI on :8000)
  - `abn-llm-gateway` (:8086)
  - `abn-security` (:8085)
- **Nginx + Let's Encrypt SSL** configured in front of the abn-core container. `https://api.abnplatform.com/health` returns `{"status":"ok","env":"production"}` over TLS.

**Verification:** the public health endpoint is reachable, the four containers are up, and `api.abnplatform.com` resolves to `46.225.165.62`.

## Batch 7 — MarketplacePage wired to real backend (DONE 2026-05-25)
**Prompt given:** Wire MarketplacePage to the real `Connector` backend — no new models or migrations (the `Connector` table already IS the tenant↔system join). Extend `client.ts` with `disableConnector` + `last_synced_at`; rewrite MarketplacePage to read from the live DB rows instead of a local Set; replace the "Kompatibel" badge with status pills (Aktiv / Inaktiv / Kompatibel); 8 new backend tests; all 1010 existing tests must still pass.

**Result:**
- Engineering rule #1: greps confirmed `initiateConnect` and `getConnectors` already existed; `disableConnector` did not. Extended what existed, added only what was missing — no duplication of `deleteConnector` (which is a one-way soft delete; PATCH `disableConnector` is bidirectional and supports re-enable).
- Caught FOUR bugs in the spec before writing code:
  1. Spec said `initiateConnect(tenantId, item.id)` — but `item.id` is a marketplace agent id like `'invoice-audit'`, not a connector type. Fixed to `initiateConnect(agent.requires, tenantId)`. Standalone agents (`requires=null`) render an "Inkluderad" pill, no install button.
  2. Spec named the response field `oauth_url` — the backend returns `connect_url`. Used the real field name.
  3. Spec called the new interface `ConnectorRow` with `id: string` — the existing `Connector` interface uses `id: number` matching the SQLAlchemy Integer PK. Extended the existing interface with `last_synced_at` rather than creating a duplicate.
  4. Spec said `POST /api/connectors/{type}/connect?tenant_id=…` (query param) — the existing backend takes `tenant_id` in the body. Kept the body form (Engineering rule #1: don't break the live contract).
- Frontend: `Connector` interface gains `last_synced_at: string | null`; `getConnectors(tenantId?, activeOnly?)` extends without breaking existing callers (both `() => getConnectors()` wrappers in ConnectorsPage + SettingsPage updated); `disableConnector(id, enabled=false)` + `enableConnector(id)` PATCH the row (pair of explicit helpers per Batch 7 contract; the `disableConnector(id, true)` form still works).
- MarketplacePage: installedMap derived from live `Connector` rows (`activeOnly=false` so disabled rows show "Inaktiv"). Per-connector-type loading spinner + inline error string. Optimistic invalidation on success. "Installera" → initiateConnect → `window.location.href = res.connect_url` (Nango OAuth redirect, per spec — popup form is in ConnectorsPage if we want it later).
- Backend tests: 8 in new `tests/test_marketplace_wire.py` — grouped by FLOW (empty → install → upsert idempotency → disable → re-enable → list → soft-delete → live-feed regression) rather than by route. Deliberately overlaps the route-level tests at the boundaries; the docstring explains why so future maintainers don't dedupe blindly. One test hit a `gps` connector that isn't in the YAML catalogue; switched to `hogia` (which is).
- Full backend suite: **1018 passed** (1010 + 8). Frontend: typecheck ✓, 60 tests ✓, Vite build ✓.

## Batch 8 — Migration deploy fix (DONE 2026-05-25)
**Prompt given:** Fix the migration deploy gap found in the prior audit (no script, Makefile target, compose/systemd/k8s step ran `alembic upgrade head` on the live server — startup used `create_all`, which never replays migration history). Replace `init_db()` in `main.py` lifespan with `alembic upgrade head`; document a one-time `alembic stamp head` on the Hetzner DB; fix the aspirational sentence in `docs/deployment.md`. Full test suite must stay green.

**Result:**
- `backend/main.py` lifespan: removed `from database.init_db import init_db` (line 20) and the `init_db()` call (line 93); replaced with `Config("alembic.ini")` + `command.upgrade(alembic_cfg, "head")`. The `init_db()` function itself stays — `scripts/setup.sh` still uses `python -m database.init_db` for fresh local dev installs.
- `backend/sidecar_entry.py`: comment update to reflect the new startup path + note that the PyInstaller bundle must include `alembic.ini` + `alembic/versions/` (separate batch when the desktop installer is next rebuilt).
- `CLAUDE.md` new section `## Production DB — one-time stamp (MANUAL — already done or do now)` with the exact `ssh root@46.225.165.62` → `cd /opt/abn/backend` → `docker compose exec abn-core alembic stamp head` → `alembic upgrade head` sequence and why a stamp is needed (Hetzner DB was bootstrapped via `create_all`, so `alembic_version` is empty/absent; running `upgrade head` against it without stamping first would try to recreate every existing table and fail).
- `docs/deployment.md` line 260: aspirational "Database migrations run automatically on startup" replaced with accurate text covering both the every-startup `alembic upgrade head` and the first-deploy `alembic stamp head` precondition.
- Verified by grep that no test imports `main.app` — every test builds a fresh `FastAPI()` and overrides `get_db`. The lifespan change is invisible to the test surface. Suite stays at **1018 passed, 84.35% coverage**.
- Flagged but not changed (separate batch): `Config("alembic.ini")` is CWD-relative; works in Docker (`WORKDIR=/app`) and systemd (`WorkingDirectory=/opt/abn/backend`); the PyInstaller desktop bundle does NOT bundle alembic files, so a fresh desktop install would fail on first run. Fix is a `datas=[…]` line in `abn-core.spec`.

## Batch 9 — Full automation chain (DONE 2026-05-25)
**Prompt given:** Wire the full Friday→Monday flow. Connector activate must trigger DNA → Process Graph → Agent creation automatically. Add an AgentScheduler that runs OPERA on the blueprint cron. Add plan-based agent limits (starter=2 / professional=25). Extend Observer `CONNECTOR_RESOURCES` to all 18 catalogue types. 15+ new tests. All 1018 existing must still pass.

**Result:**
- Engineering rule #1 flagged FIVE spec-vs-reality conflicts before code:
  1. Quality threshold: spec said "≥ 0.6", `process_graph/runner.py` already calibrates at AUTO=0.85 / VERIFIED=0.70 with an existing `agent_decision` field. Used the existing field per rule #2.
  2. `DNAPhaseOrchestrator(db, tenant_id)` doesn't exist — the real signature is `(db, config: DNAPhaseConfig, progress_callback)`. Constructed a real config.
  3. `ProcessGraphRunner(db, tenant_id)` arg order is wrong — real is `(tenant_id, db, domain, mode)`. Used the real signature.
  4. All three chain calls are `async` — wrapped them in an `asyncio.run` shim so the sync FastAPI BackgroundTasks queue can drive them.
  5. Hardcoding `CONNECTOR_RESOURCES` for 18 types would duplicate the YAML catalogue. Built `_build_connector_resources()` that reads YAML at module load + a `_seconds_to_cron` translator (rule #2).
- `api/routes/connectors.py`:
  - `BackgroundTasks` injected into `initiate_connect`.
  - New `_trigger_automation_chain(tenant_id, connector_type)` runs DNA → Graph → Agent end-to-end, each step in its own try/except so a fresh-install tenant with zero events fails defensively. Owns its own `SessionLocal()` since the request scope is closed by the time the task fires.
  - `_schedule_automation_chain` adapter wraps the async chain in `asyncio.run` for the BackgroundTasks queue.
- `agent_engine/scheduler.py` (NEW): `AgentScheduler` mirrors `ObserverScheduler`. Daemon thread, 60 s tick, `croniter`-based, `_running_tenants: set[str]` for one-run-per-tenant. Reads `agent.blueprint["schedule"]["cron"]` (also accepts legacy string form). Wired into `main.py` lifespan after the Observer scheduler.
- `api/routes/agents.py`: new `_PLAN_AGENT_LIMITS` dict (starter=2 / professional=25 / enterprise=10000), `get_tenant_plan_limits(db, tenant_id)`, `_count_active_agents(db, tenant_id)`, `enforce_agent_limit_or_raise(db, tenant_id)`. Gate added to `POST /api/agents/generate-from-process` — only on the non-idempotent path (existing agent for the process graph still returns 200 with `created=false`).
- `observer/scheduler.py`: `CONNECTOR_RESOURCES` rebuilt at import time from YAML. Outbound-direction YAMLs (`slack_delivery`, `sendgrid`) yield empty lists. Legacy hardcoded `gps` preserved as fallback (no `gps.yaml`).
- Two real bugs caught + fixed during test runs:
  1. `_is_due` had a wrong `_DUE_WINDOW` upper-bound check that rejected late-firing cron windows. Removed; `nxt <= now` is the right condition.
  2. `_is_due` short-circuited to True on `last_run_at=None` BEFORE validating the cron, so a malformed cron with no last-run was treated as "due now". Moved `croniter.is_valid` up.
- 18 new tests in `tests/test_automation_chain.py`: YAML coverage (3), automation chain (3), AgentScheduler (6), plan limits (5), regression (1).
- Full backend suite: **1036 passed** (1018 + 18), 84.45 % coverage.

## Batch 10 — DNS + public API endpoint (DONE 2026-05-25)
**Prompt given:** Wire DNS + public API endpoint for `www.abnplatform.com` and `api.abnplatform.com`. Add `vercel.json` with apex→www 308 redirect, security headers on `next.config`, `public_api_url` + `allowed_origins` in backend Settings, CORS reading from settings, `/health` returning `public_url`, replace any hardcoded localhost / `abn-nine.vercel.app` references. Additive only — no router or middleware re-architecting. 1036 tests must stay green and `npm run build` must produce 33 pages.

**Result:**
- Engineering rule #1/#2 flagged FOUR spec-vs-reality conflicts before code:
  1. `Settings` already has `cors_origins` + `cors_origins_list` (config.py:45 / 119). Spec said to add a parallel `allowed_origins` — **kept `cors_origins`** as the single source of truth (rule #2). Extended its default to include `https://www.abnplatform.com` + `https://abnplatform.com`.
  2. `main.py:181` CORS middleware already reads `settings.cors_origins_list` — Task 4 was essentially already done. Added an empty-list `["*"]` fallback for dev safety per the user's intent.
  3. Spec said `next.config.js`; the real file is `next.config.mjs` (ES module). Edited in place.
  4. Spec Task 6 said "replace any hardcoded `localhost:8000` / `localhost:3000` / `abn-nine.vercel.app` in landing" — **0 matches** in the grep. Nothing to fix.
- `landing/vercel.json` (NEW): apex→`www` 308 via `redirects[]` with `has.host` matcher. `$schema` set so Vercel's editor validates.
- `landing/next.config.mjs`: kept `reactStrictMode: true`; added `async headers()` returning `Strict-Transport-Security` (2 years incl. subdomains + preload), `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=(), geolocation=()`.
- `backend/core/config.py`: NEW `public_api_url: str = ""` field. `cors_origins` default extended with the two production origins.
- `backend/.env.example`: `PUBLIC_API_URL=https://api.abnplatform.com` block added next to `ABN_ENV`. `CORS_ORIGINS=` line extended with the two production origins. (No `ALLOWED_ORIGINS` env var — `cors_origins` is the canonical name; comment explains why.)
- `backend/main.py` CORS: now reads `_cors_origins = [o for o in settings.cors_origins_list if o]` and uses `_cors_origins or ["*"]` so an empty env doesn't 502 the dashboard in dev.
- `backend/api/routes/health.py`: `/health` response gains `"public_url": settings.public_api_url`. Additive only.
- `CLAUDE.md`: new `## DNS & Domains` section with the host→target table, manual registrar steps (Vercel A `76.76.21.21`, CNAME `cname.vercel-dns.com`, Hetzner A `46.225.165.62`), and the manual Vercel project step (add both domains under `abn-nine`).
- Backend suite: **1036 passed** (unchanged). Landing: `✓ Compiled successfully`, `✓ Generating static pages (33/33)`, zero errors / warnings.

## Batch 11 — NEXT
Fortnox end-to-end (needs org number + API key from Jacob).

## Batch 12
Branch-protection hardening (manual GitHub step — listed in CLAUDE.md under `## Branch Protection`).

## Batch 13
Bundle `alembic.ini` + `alembic/versions/` into the PyInstaller `abn-core.spec` so the Tauri desktop sidecar can run `alembic upgrade head` on a fresh install (Batch 8 follow-up).

## Manual prereqs for Stripe to go live
1. Stripe Dashboard → create two Products + Price objects (Starter free / Professional €299/mo recurring) and copy their `price_...` ids.
2. Stripe Dashboard → Developers → API keys → copy the `sk_test_...` (or `sk_live_...`) secret key.
3. Stripe Dashboard → Developers → Webhooks → add endpoint `https://api.abnplatform.com/api/billing/webhook`, subscribe to `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`. Copy the `whsec_...` signing secret.
4. Set all four env vars on the Node (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PROFESSIONAL_PRICE_ID`, `STRIPE_STARTER_PRICE_ID`). The billing routes return 503 until they are set.

## Blockers
- Fortnox: waiting for org number + API key from Jacob
- ~~DNS: Proton mail propagation (hello@abnplatform.com)~~ — **resolved 2026-05-25 (Production deploy event)**
- ~~MarketplacePage was a stub (local Set, no DB writes)~~ — **resolved 2026-05-25 (Batch 7)**
- Google Cloud OAuth: ~~waiting for mail-domain verification~~ → unblocked (Proton mail-domain verified); add the OAuth callback URL when ready

## Technical debt
- 7 duplicated package dirs (`agent_evolution/agent_evolution/`, `consistency/consistency/`, `delivery/delivery/`, `gdpr/gdpr/`, `inquiry_engine/inquiry_engine/`, `onboarding/onboarding/`, `opera/opera/`, `output_schema/output_schema/`) — cleanup sweep deferred
- Two OPERA runners coexist (`agent_engine/opera/runner.py` production, `opera/integrated_runner.py` legacy) — consolidation deferred
- `AgentBlackboard` table missing — reads degrade gracefully to empty list
- Blueprint version history missing — `POST /api/agents/{id}/rollback/{v}` returns 501
- `services/abn-security/` (Go) not compiled in this environment — CI job covers it
- Many modules use deprecated `datetime.utcnow()` — cosmetic warnings only

## ABN Innovation Philosophy

Build ABN like the greatest innovators built their work:

- **Ada Lovelace** — first to see what machines could truly become
- **Alan Turing** — redefined intelligence, built the foundation of computing and AI
- **John von Neumann** — architected how modern computers think and store
- **Steve Jobs** — made complexity invisible, obsessed over experience
- **Bill Gates** — saw software as infrastructure for the world

Rules:

- Never ask "does this exist?" — ask "should it exist, can we build it better?"
- If a module ABN needs doesn't exist: use Python, Linux, open source — adapt it, make it ABN's own
- Every feature must be 100,000x better than competitors — or don't ship it
- Think in platforms, not features. Think in decades, not sprints
- ABN has no real competitor — because no competitor thinks like this

## Rules
- Always read CLAUDE.md first
- Grep before building — no duplication (Engineering rule #1 in CLAUDE.md)
- All prompts to Claude Code in English; code/comments/identifiers in English (rule #3)
- Act as 4 engineers: Architect, Engineer, Reviewer, Optimizer
- One batch at a time, green build before next (rule #8)
- After every batch: update this file + CHAT_LOG.md and push to GitHub (see `## Session Management` in CLAUDE.md)

## OPEN ITEMS
Unfinished sub-tasks / open questions live here (referenced by CLAUDE.md §4.2 PRE-/clear HALT PROTOCOL). Empty = nothing pending.
- Guardrails-config PR (chore/guardrails-upgrade) awaiting Jacob review — do NOT auto-merge; Jacob checks the 6 verification exit codes + the new CLAUDE.md §4.1/4.2/4.3 + deny[] for false positives before merging.
- Manual (Jacob, GitHub UI): add "abn-security — Go build & test (45a)" as a required status check in branch protection.
- Version lockstep drift (future OPS batch): the canonical version is hand-mirrored in `frontend/src-tauri/tauri.conf.json` + `backend/core/config.py` `abn_version`; the two package.json files are drifted at 1.0.0. (Batch 74 removed the landing `DownloadView.tsx CURRENT_VERSION` mirror — the download page now reads the version dynamically from the public release, so that copy is no longer a drift risk.) A later batch could add a CI/pre-commit assert that the 2 remaining mirrors match.
- **HUMAN STEP (Batch 74): mint `MIRROR_PAT`** — a fine-grained PAT with `contents: write` on `abn-systems/abn-releases`, stored as a repo secret named `MIRROR_PAT` on `abn-systems/ABN`. Without it, `mirror-release.yml`'s public-upload step fails loudly (by design). The mirror reads the private repo with the default GITHUB_TOKEN; only the public push needs the PAT.
- Batch 74 follow-up: if `release-bandwidth-guard.yml` ever opens its "Release bandwidth approaching limit" issue, plan a CDN / object-store batch for installer hosting. The guard is a conservative proxy (no live GitHub bandwidth API), so treat the alert as a trend signal.
- Stale CI gate label (flagged by Batch 75b — NOT fixed, out of test-only scope): the ci.yml backend job `name: "Backend — 1518 tests"` and the matching branch-protection required-check key both say 1518, but the real suite is 1535 (Batch 46 added +13, etc.). The name is cosmetic — pytest never asserts the count — so CI stays green; the recommended permanent fix (per CLAUDE.md's recurring count-drift note) is to rename the job to a count-free `"Backend — tests"`, which Jacob must change in ci.yml AND the GitHub Settings required-check key in lockstep. Until then leave both untouched so the check keeps reporting. (The Batch 74 freeze-clock open item is now RESOLVED — see CHANGES.)
- Batch 64 (INFRA Task 2 CI gap-fill) follow-ups: (1) the count-free rename of the required `Backend — 1518 tests` job — SAME item as the line above; Batch 64 made the ci.yml *comment* count-free but deliberately did NOT touch the job name or the branch-protection key (lockstep rename is its own batch). (2) [SUPERSEDED by Batch 66a — CodeQL was REMOVED (GHAS not enabled); re-adding it, and revisiting Go coverage, is gated on enabling GHAS — see the dedicated "Re-add CodeQL" item below]. (3) Wire ZAP → SARIF to the Security tab (`security-dast.yml` currently surfaces via the `zap-dast` workflow artifact; native SARIF needs a converter — deferred to keep the advisory DAST job hermetic/never-red).
- Batch 65 (Task 3 adversarial gap-hardening) — KNOWN LIMITATION (surfaced, NOT a bug, NOT shipped red): the gateway regex PII scrubber (`services/abn-llm-gateway/.../pii_scrubber.py`) is plaintext-only by design, so obfuscated PII (base64 / unicode-homoglyph / IBAN) in a NON-tokenised free-text field reaches the provider in `redacted`/`full` mode — it is neutralised only in `no_data` mode (the abstractor drops all values to types). NOT a No-Data-guarantee violation: `no_data` tenants and every tokenised field are fully protected in all modes; tested in `TestObfuscatedPII`. Future hardening options for Jacob to weigh: (a) decode-then-scrub (base64/homoglyph-normalise before the regex pass), (b) an IBAN pattern, (c) document `no_data` as the recommended default for high-assurance tenants. Low urgency — defence-in-depth, not an open hole.
- Re-add CodeQL if/when GHAS (GitHub Advanced Security) is enabled — removed in Batch 66a because its SARIF upload requires GHAS (a PAID add-on on private repos, not enabled), so it was red on every PR. SAST stays covered free by Semgrep (`security-sast.yml`). The `codeql.yml` workflow is preserved in git history (fully reversible); re-adding would also revisit CodeQL Go coverage for `services/abn-security`. Future OPTION, not a todo.
- Batch 66 Task 4 — **COMPLETE** (Part 1 #75 + Part 2 PR #76, branch `feat/batch-66b-shield-runtime`). Active layer shipped: always-on monitor on the EXISTING scheduler tick, attack-SUCCESS detectors (reuse `ShieldGuardian.run_checks()`, 3 originals untouched), conservative confirmation gate, `deliver_alert` reuse (no new notifier), metadata-only `shield_alerts` persistence, fail-OPEN-on-monitor/detector-error / fail-SAFE-on-confirmed-breach monitor that SETS `Agent.quarantined`. Backend 1583→1595 (+12); SQLite migration verified, Postgres half on CI. DEFERRED — **INVESTIGATED + CONFIRMED PERMANENT** (Discovery pass, Batch 69 Phase 0.5; per Jacob): **B1c runtime leak-detector — NOT buildable as a Shield quarantine detector** (No-Data leaves no metadata artifact by design); future defense-in-depth = a separate fail-CLOSED gateway residual-PII guard OR an advisory hygiene check, neither a quarantine detector. Do NOT build/force it as a Shield detector. (Discovery detail so future Phase 0.5 need not re-investigate: the 3 candidate seams all fail — `LLMGatewayLog.had_unmapped` detects LLM token *hallucination*, not leakage; `abn_llm_calls.{raw_values,customer_data,pii}_sent` are hardcoded `0` *guarantees*, not measurements; Observer `cycle.py` `sent_to_llm=0`/`sent_outside=0` are hardcoded constants; the Batch-65 canary/no-leak check is a *pre-merge pytest assertion*, not a runtime signal. A genuine leak leaves no durable aggregate artifact by design — the exact opposite of the 3 working success-detectors, each of which keys off a structurally-impossible persisted row. Every path to a real runtime measurement breaks a guard: hold/persist raw data, read `Finding` payload columns, or add in-gateway/in-agent instrumentation. The standalone #81 doc PR was closed; this note is the canonical record.)
- Batch 66 Phase D — gate-coverage DECISION (resolved for the main path): added **gate 3** at the canonical manual-run entry `POST /api/agents/{id}/run` — a quarantined agent's manual run is refused with 409 at the safe boundary (ALL tiers, not just tier-3). So: gate 1 (scheduler) stops scheduled runs, gate 2 (tier-3 pre-flight) refuses privileged writes, gate 3 (manual-run API) refuses manual runs. ~~STILL DEFERRED~~ **CLOSED by Batch 72b**: the conversational `/instruct` `run_now` path is now gated (via the unified create-gate at the top of `OPERARunner.run()` + a UX wrapper), and the single unified gate inside `OPERARunner.run()` is built (Option A). NOTE: 72b's unified gate is for `pending_human_approval`; the Batch-66 *quarantine* gates remain replicated as before. A future consolidation of quarantine ALSO into the run() boundary is a clean follow-up (not forced — quarantine's existing placement is deliberate + proven).
- **Batch 67b — Task 6 Part 2 (the autonomous write path) — DONE (PR open, awaiting Jacob review; Task 6 COMPLETE).** Shipped: the all-conditions `mind_autonomy.autonomy_allowed` gate (platform ∧ tenant ∧ approved policy ∧ clamp ∧ Accept(B)-post-change ∧ not-quarantined, fail-CLOSED), the bounded `apply_autonomous_threshold_adjustment` write (out-of-band = refusal; RollbackRecord committed ATOMICALLY with the AgentSettings write via the existing `_handle_set_threshold`; metadata audit in `RollbackRecord.new_state`; reversible via new `_rb_mind_adjust_confidence_threshold` in `_ROLLBACK_DISPATCH`), the `MindAgent.run_autonomous_adjustments` consumer (weekly runner only, tightens-only, safe direction), and `mind_runner.py` wiring (fail-soft). **Both Part-1 design problems RESOLVED:** (1) Accept(B) ceiling → new public `ABNMetaAgent.accept_candidate(blueprint_dict)` builds the POST-change blueprint via `_build_blueprint` + runs `accept()` over the 8 V_GLOBAL invariants, required True before any write (admin opt-in cannot override the constitution); (2) "dry-run an adjustment" → new small `simulate_threshold_adjustment` (before→after + clamp + honest directional projection, `effect_modelled="directional_only"`), NOT a reuse of `simulate_run`. No new schema/migration (reuses RollbackRecord/AgentSettings/Tenant.policy). 19 tests (refusal proofs + fail-closed + Accept(B) ceiling + default-off), backend 1617→1636, 0 regressions.
- **Batch 68b — Design system Part 2 (the breadth) — DONE (PR open; Batch 68 COMPLETE).** Transcribed the 48 extended motifs (`icons/extended.tsx`) + 10 principles (`icons/principles.tsx`) into the SAME registry (appended to `ABN_ICONS`, not refactored) → **80 marks total** (12 core + 48 extended + 10 principle + 10 status); + a dev gallery `pages/IconGalleryPage.tsx` at `/dev/icons` (additive route). Palette constants (no per-icon hex), kebab keys unique-verified, 80-mark smoke-render test green. typecheck ✓, vitest 74→75, build ✓. No backend/DB. **Remaining design-system follow-ups (small, NOT blocking):** (a) expose `quarantined` on the frontend `Agent` type in `api/client.ts` so `agentStatusToIconName` reads it without an explicit arg; (b) optionally adopt `<AbnIcon>` in existing views where ad-hoc status text/illustrations are rendered (pure UI polish). Status-enum note carried from 68a: "Tänker"/"Väntar" have no single backend status string — mapped from transient run-state tokens when supplied (don't invent backend statuses).
- Doc/test naming reconciliations surfaced in 67a (cosmetic, not fixed): the real Mind privacy test is `test_mind_no_customer_data_in_report` (the Batch-67 prompt + discovery called it `test_mind_no_customer_payload_in_metrics`); and `alembic check` reports pre-existing repo-wide autogenerate drift (TIMESTAMP↔DateTime, REAL↔Float, unique-index-vs-constraint across ~19 tables) — NOT introduced by 67a (the new `mind_autonomy_policies` table is absent from the diff) and NOT what `migrations-dual.yml` runs (it runs `alembic upgrade head`, which is green). A repo-wide migration-autogenerate-alignment pass is a possible future OPS batch.
- **Batch 69 — Task 5 Pulse hardening DONE** (PR open; EXTEND Batch 25, no rebuild): Issue auto-close/recovery, flap suppression (`fail_threshold` dial), unverified-URL advisory (`verified` flag — fortnox true, quinyx/hogia false), cron built-in-`GITHUB_TOKEN` (no PAT sprawl). Advisory-only + fail-OPEN + metadata-only preserved; no new table (auto-close/flap read existing `PulseResult` history). DEFERRED until LIVE connectors exist (post Task 1 — the integration layer): (a) Pulse coverage expansion beyond the 3 hardcoded TIER-3 connectors; (b) dynamic `pulse:`-block-driven coverage; (c) verify the quinyx + hogia health URLs and flip their YAML `verified: true` (today they are placeholder guesses, advisory-only). B1c No-Data-leak detector remains PERMANENTLY deferred (see the Batch-66 Task-4 OPEN ITEM above; standalone #81 doc PR closed/folded into this batch's Phase 0.5).
- **Batch 70 — image-build CI flake RESOLVED** (PR open; OPS hygiene): the `docker` ("Build & push images") job in `ci.yml` was red on main (anonymous `moby/buildkit` Docker Hub pull timing out at buildx bootstrap + `fail-fast:true` cascade). Fix = `fail-fast: false` on the matrix + `driver: docker` on `setup-buildx-action` (removes the anonymous pull; no secrets). Advisory job (NOT a required check). DEFERRED-UNLESS-NEEDED: Docker Hub auth via `docker/login-action` + `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` repo secrets — only warranted if the base-image (python/nginx/node) pulls inside the Dockerfiles ever start rate-limiting (the buildx-bootstrap pull, the actual failure, is gone with `driver: docker`).
- **Batch 71a — generator ↔ Accept(B) reconciliation DONE** (PR open; security-critical prerequisite). `generate_blueprint` now declares the constitutional posture in `policy_constraints` (single-source `_constitutional_policy_constraints(tier, policy)`, every line mapped to a real enforced mechanism, tier-aware, re-derived after AVM). Generated ANALYZER/T1 + PLANNER/T2 blueprints now pass Accept(B) with 0 violations (verified empirically + new regression tests — the Batch-43 gap where only hand-built fixtures were tested). Defuses the latent sweep landmine. CLAUDE.md Batch-43 drift annotated honestly. Suite 1644→1647. **NEXT: Batch 71b** — the synchronous fail-CLOSED Accept(B) sign-time gate (now safe: legitimate blueprints pass). Do 71b only AFTER 71a merges + is proven.
- **Batch 71b — Accept(B) synchronous fail-CLOSED sign-time gate DONE** (PR open). The gate sits in `generate_blueprint` after 71a's `policy_constraints` re-derive + before `sign_blueprint` (single choke: trigger auto/verified, orchestrator, API); reuses `ABNMetaAgent.accept_candidate` (no new verifier, 8 invariants unchanged). FAIL-CLOSED: violation OR checker error → `AcceptBRefusedError` (new, subclasses `BlueprintValidationError`) → not signed/persisted/run. Defense-in-depth (sweep/manual/Mind) unchanged. Tests +4 (allow-legit T2, block-violator T1, fail-closed-on-error T4, auto-path-no-run T3). Suite 1647→1651. **Accept(B) sign-gate COMPLETE (71a+71b).** Remaining Agent-Creation items unchanged (see next bullet).
- **Agent-Creation-Intelligence — remaining items (from the Discovery):**
  - **Prio 2 / 72a — DONE (PR open):** AgentNeedScore always-value decision (wires the computed ROI) + report-only fork. `decide_agent_need` → {FULL_AGENT / MONITOR_DRAFT / INSIGHT_REPORT / OBSERVED_ONLY}; report outcomes generate tier-1 READ_ONLY agents (reuse tier-ceiling); OBSERVED_ONLY surfaced. Floor + Accept(B) intact; no gate/schema/run-path change. Suite 1663.
  - **Prio 2 / 72b — DONE (PR open; Prio 2 COMPLETE).** Risk-scaled human create-gate. DECISION (Jacob signed off): **OPTION A — ONE unified gate at the TOP of `OPERARunner.run()`** (`_raise_if_pending_approval`), covering all 5 run entry points at the single choke; per-entry-point wrappers (scheduler skip, manual-/run 409, /instruct message) are convenience only. New `agent_engine/create_gate.py` `requires_human_approval` (tier-3 / EXECUTE_CHANGE ⇒ pending, fail-CLOSED, No-Data) sets `Agent.pending_human_approval` at sign-time. `PendingApprovalError` → run recorded `awaiting_approval`. Schema: 4 columns (`pending_human_approval`/`pending_reason`/`approved_by`/`approved_at`, `sa.false()`), migration `c5e9b1a7f3d2` (revises `b8d3f1a26c47`, dual-DB, SQLite-verified). `POST /api/admin/agents/{id}/approve-activation` (NODE_ADMIN, tenant-scoped, idempotent, ABNActivityLog audit) + `GET /agents/pending-activation` queue. **V1 reality: NO generated agent is create-gated yet** (build_blueprint caps tier at 2); the gate is the forward-looking V2 boundary. **ApprovalRecord deviation (documented):** used ABNActivityLog (its proposals FK is dual-DB-unsafe for a non-proposal activation); Proposal-queue surfacing is a future UX add. Tests 27+2, suite 1663→1692, 0 regressions.
  - **Prio 3 (later):** hard minimum-conformance FLOOR (conformance is only 25% of the blended quality — a high-volume/low-fitness graph can still cross the threshold) + a **dry-run-before-deploy** gate at creation (`simulate_run` exists but isn't wired into `generate_blueprint`).
- **OPERA + Blueprint-creation integrity — Discovery DONE (read-only, authoritative). Engine-integrity queue (close in order, Discovery before build, one batch each):**
  - **[73 DONE — merged #89] run() status honesty.** `OPERARunner.run()` checked only Observe+Execute → a FAILED run persisted as "success" (Plan error swallowed → empty plan → masked success; Act failure read empty `{}` output → "success"). Fixed surgical: run() now checks Plan + Act status, fail-CLOSED → "failed" with a metadata reason; legitimately-empty plan stays a truthful success (the crux: `_phase_plan`'s only non-"success" path is its except → "partial"); Reason graceful-degrade + Observe/Execute untouched. No schema. Tests +9 (`test_opera_status_honesty.py`), suite 1692→1701.
  - **[74 DONE — PR open] P1(b) Build 1: Critic + End-State Evaluator on the live path.** Wired Dual-Brain Critic + ESE into `run()` as a fail-CLOSED pre-delivery verify stage — CALLED the functions, did NOT flip to the AAEA executor (avoids double-run / run_id collision / dashboard breakage). `_phase_act`→build-only; new `_phase_verify` + `_phase_deliver`. Crux: the critic's POSSIBLE_NAME heuristic false-positives on ABN's own bi-capitalised prose → categorise issues (strong-PII/contradiction/policy/unknown = fail-CLOSED block; POSSIBLE_NAME + confidence-vs-tier = soft flag). Verdict→honest status (extends 73): block→failed (not shipped), ESE flag 0.50-0.70→delivered+needs_review, ≥0.70→delivered; empty-but-correct still ≥0.50→success. No-Data (abstract-only LLM critic). No schema. Tests +12 (`test_verify_onpath.py`), suite 1701→1713.
  - **[75 DONE — PR open] MND: ABN Failure Taxonomy v1.** `core/failure_taxonomy.py` — single-source `FailureClass` (27 canon classes) + `FailureSpec` registry with the six fields (severity/retryable/requires_human/auto_pause_weight/safe_to_show/safe_to_feedback) + truthful reference-map (`maps_to` real token or None + `emitted_by`). API get()/iter_all()/all_classes()/is_known(). CATALOG ONLY — no emitter rewiring, no live-path change, no schema; weights are calibratable placeholders. The shared failure vocabulary RAL/AgentHealth/Safe Mode/Supervisor/RunStatusReducer will read. Tests +20 (`test_failure_taxonomy.py`), suite 1713→1733.
  - **[76a0i DONE — PR open] real finding-production (context-threading + permanent sim-guard).** Two Discoveries found the live path produced 0 real findings (fresh-context-per-step isolation in `_run_capability`) + `read_*` simmed unconditionally (no adapter wired). Fixed: P1 thread ONE shared context (read→analyze→report collaborate) + read findings once (no double-count); P2 PERMANENT sim-guard (`capabilities/_source_guard`, single source) — no adapter + `allow_simulated_findings` False (prod default) → genuinely empty + `needs_connector` (never fabricated); adapter error + sim off → truthful failure; P3 `needs_connector` reason (observe_no_data), truthful empty success. Flags: `enable_capability_context_threading`=True, `allow_simulated_findings`=False. PERMANENT RULE: no fabricated/simulated finding on any prod/customer surface. P4 (PII-on-findings) deferred to 76a. Tests +10 (`test_finding_production.py`), suite 1733→1743. **Real adapter injection into OPERA is itself target (Spår 3 DNA) — prod runs are truthfully empty until a connector is wired; sim is dev-only behind the flag.**
  - **NEXT — 76a (RAL finding-shape, now meaningful): critical-field values + domain→FIELD_MAPS resolver + informational RAL stamp + `Finding.attested` write-back.** RAL `field_attestor`/`cross_reference_checker`/`attestation_report` are built but NOT on the live path; `Finding.attested` is structurally False (no writer; `findings_persistence.py:136` reads `raw.get("attested", False)`) → `attestation_rate_pct` ~0 for ALL agents. 76a: (a) capabilities emit critical-field VALUES + the natural key (+ `source_a`/`source_b` for cross-ref); (b) **explicit domain→FIELD_MAPS resolver** (real domains like `carrier_billing` don't match `FIELD_MAPS` keys → `domain_key=None` → silent skip); (c) run attest_fields/cross_reference/build_attestation_report informational in `_phase_verify` + **write the attestation result back onto each finding** so the metric reflects reality; (d) **fold in P4 PII-on-findings reconciliation** (RULE 3 recurse into findings vs Critic PII scan). Then **76b** = RAL min()-fold ENFORCEMENT (hard-block when `blueprint.ral_required_fields` non-empty + attestation fails). Then **P2a** conformance floor + VariantConformanceMap → **P2b** SimulationGate.
  - **P2(a): hard conformance/fitness floor.** `calculate_quality_score` (`process_graph/statistics.py:66-77`) blends fitness at only 25%; volume+confidence+algorithms reach ~0.72 with fitness=0.0 → crosses VERIFIED 0.70 → an acting agent from a graph whose model doesn't fit. No hard minimum-fitness gate.
  - **P2(b): dry-run-before-deploy.** `simulate_run` exists but is never called in `generate_blueprint` (confirmed by reading the full create path).
  - **Then: the big DNA/Agent-OS Discovery map (areas A–G)** — DNA/Observer-modes/Event-Envelope/case-key/no-loss; 6 graph levels/GraphConfidence/Work-Units/Genesis Package; BrainProfile/ToolFit/Verification/Simulation compiler; runtime supervisor/Trust-Kernel/Coordinator/Jurisdiction; local Control-Plane SQLite registers/Health; Evolution/Shadow-mode/Kill-switch/invariants; Tauri control room — one area at a time, Discovery before build.

## CHANGES
Operational-config change log (newest first).
- Batch 75b (test-hygiene, TEST-ONLY) — froze the clock in the 6 calendar-flaky tests: test_anomaly_trend.py::{direction_improving, alert_triggered} + test_mind_agent.py::{report_generated_correctly, suggestion_rollback_spike, suggestion_low_attestation, suggestions_are_swedish}. Added `freezegun==1.5.5` to backend/requirements.txt + a shared `tests/conftest.py::frozen_clock` fixture pinning `now` to 2026-06-17 (a Wednesday, mid-month); both modules opt in via `pytestmark = pytest.mark.usefixtures("frozen_clock")`. Root cause: the tests seed rows at `now - N weeks/days` and assert they fall inside a window the product computes from its own `datetime.utcnow()` read — on a Monday/month-start the seeded `now - 1 day` row drops into the previous ISO week. Product code (Batch 28 anomaly trend, Batch 30 Mind agent) is correct; only the tests were time-fragile. Proven both ways under a simulated ambient clock of 2026-06-01 (Monday + month-start = the real boundary that broke CI today): UNFIXED → exactly those 6 fail, 11 pass; FIXED → all 17 pass. `git diff main..HEAD` on backend/{agents,core,services,api} is empty — no product code touched; assertions unchanged. NOTE: the real backend count is 1535 (not 1518); the change is count-neutral (1535 collected with and without it). PR opened (do NOT auto-merge — Jacob merges this first to unblock PR #27's Backend gate).
- Batch 73 GITHUB_TOKEN chaining gap — RESOLVED. Jacob created a fine-grained PAT (Contents: read/write on abn-systems/ABN) and stored it as repo secret `RELEASE_PAT`. `auto-tag.yml`'s release step now authenticates `gh release create` with `${{ secrets.RELEASE_PAT }}` (no GITHUB_TOKEN fallback — a missing PAT must fail loudly, never silently create a Release that cannot chain the installer build). The full chain (tauri.conf.json version bump → tag + Release → build-release.yml installer build) is now automatic end-to-end. PR #25.
- guardrails upgrade applied — CLAUDE.md §4.1 continuous-save / §4.2 pre-/clear HALT protocol / §4.3 self-identity anchor; .claude/settings.json deny[] (15 durable-artifact entries) + PreToolUse Bash hook; .claude/hooks/guard.sh (extended BLOCK pattern). Guard verified: 4 BLOCKED (exit 2), 2 passed (exit 0). Config-only, no backend/services/tests/frontend/landing touched, no version bump, no ci.yml change.
