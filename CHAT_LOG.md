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

## 2026-05-27 — Batch 23 — Quinyx + Hogia write-back, DPA v1.1, connector toggles

Broadens TIER 3 EXECUTE_CHANGE from Fortnox-only to a three-system allowlist. Adds Quinyx (schemaläggning) + Hogia (ekonomi) as first-class write connectors with the same opt-in shape Fortnox already uses.

**Engineering decisions:**

- **Refactored fortnox_write.py to use shared helpers — rule #2 overrode "mirror exactly".** The spec said "Mirror fortnox_write.py pattern exactly" but Engineering rule #2 says "One source of truth per concept. When two code paths need the same behaviour, both call the same helper." Triplicating the four internal helpers (~150 lines) across fortnox_write.py + quinyx_write.py + hogia_write.py would be ~450 lines of duplicate code that would drift on every Batch 22C-style update. Created `_tier3_write_helpers.py` with `Tier3WriteBlockedError` (base) + the four canonical helpers. fortnox_write.py refactored to thin wrappers; all 22 fortnox-related tests stayed green through the refactor. Per-connector exception classes (`FortnoxWriteBlockedError`, `QuinyxWriteBlockedError`, `HogiaWriteBlockedError`) subclass the base — existing test catches still work.

- **Module-level `_nango_sync_call` thin wrapper kept per-connector.** When refactoring fortnox_write.py the first version had the GET/PUT helpers call `nango_sync_call` from the shared module directly. Three tests failed because they `monkeypatch.setattr(fw, "_nango_sync_call", _stub_nango)` — the monkeypatch couldn't intercept the call into the shared module. Fix: each connector module keeps a local `_nango_sync_call` thin wrapper that delegates to the shared helper. Test stubs still intercept; new code can use either symbol.

- **Quinyx `approved_payroll` is hard-forbidden.** Mirrors the Fortnox `paid` allowlist reasoning. Once a shift is payroll-approved, the upstream Hogia/payroll batch has already consumed it; reverting it creates a reconciliation gap. Auditor-only territory in V1.

- **Quinyx `create_absence` stores the new `absence_id` in `new_state` for rollback.** Most rollback flows restore a prior value from `prior_state`. Absence creation has no prior absence to restore — the rollback is "DELETE the absence by id". The capability captures the POST response's `id` field into `new_state.absence_id`, and `_rb_quinyx_create_absence` reads it to call `DELETE /v1/absences/{id}`. Documented inside the dispatch handler.

- **Hogia API response shape is flat, not enveloped.** Fortnox returns `{"Invoice": {field: ...}}`; Hogia returns `{field: ...}` directly per the OpenAPI spec. So hogia_write.py uses straight dict accessors (`body.get(field)`) rather than the envelope helpers. Same pattern as Quinyx.

- **DPA v1.1 is backward-compatible.** The PDF generator branches on `dpa_version` string comparison (`(dpa_version or "").lower() >= "v1.1"`). v1.0 renders the original "Endast Fortnox" bullet; v1.1 renders "Tillåtna system (v1.1)" with all three connectors listed. Existing v1.0 signatures stay valid (per the `Tier3DPASignature` unique constraint on `(tenant_id, dpa_version)`) so customers don't get logged out of TIER 3 by the version bump. AdminPage State C shows an "äldre — kan signeras om" badge when `dpa_version < current_dpa_version` so the customer is prompted to re-sign for the broadened write surface.

- **`_rb_resolve_connection_id` is the single source of truth for the column-vs-prior_state policy.** Six new rollback handlers all need the same column-first + prior_state-fallback lookup (Batch 22C carry-over). Pulled it into one helper rather than copy-pasting the same 7 lines six times. Existing 3 Fortnox handlers still inline the lookup — refactoring them was out-of-scope for this batch and they're harmless duplication of a 7-line snippet.

- **`Tier3ConnectorMatrix` reused in both State B + State C.** State B ("not signed yet") shows the matrix as "what would be activated after signing". State C ("signed") shows it as "what is live right now". Same component, different framing label. Bullet list trimmed to the three universally-applicable items (max-1-write, rollback, No-Data) since the per-connector specifics are now in the matrix.

**Verification:**

- Backend pytest: **1214 passed** (1202 + 12, zero regressions, 112 s)
- Go suite (`golang:1.22-alpine`): all 7 packages green
- Frontend: typecheck ✓, 60 tests ✓, Vite build ✓
- Landing Next.js build: 33 static pages ✓

**Manual pendings (unchanged from Batch 22C close-out):**

- Hetzner Postgres `alembic stamp head` still pending — production restart will fail until the operator runs the two commands documented in CLAUDE.md `## Production DB`. Head expected: `10f575a96ce4 (head)`. (No new migration in Batch 23 — Quinyx + Hogia reuse the existing `RollbackRecord` table.)
- GitHub branch-protection rule key: flip `Backend — 1202 tests` → `Backend — 1214 tests` in the Settings UI.
- `SESSION_MIRROR_TOKEN` still expired — re-run the public mirror sync to `abn-session-memory` repo manually after push.

**What's next — Batch 24 (preview):**

TIER 3 telemetry — the production outcomes feed:
- Per-tenant aggregate counters: writes_attempted / writes_succeeded / writes_rolled_back per (connector, agent, week)
- AdminPage "Tier 3 outcomes" card surfacing the counters
- Alert thresholds (e.g. > 10 % rollback rate triggers operator notification)
- Optional opt-in telemetry channel ABN ↔ ABN Platform AB — aggregate-only, no per-tenant or per-write identifiers — so Customer Success can spot deteriorating tenants before they churn (this would require its own DPA addendum bump to v1.2)


## 2026-05-26 — Batch 22C — TIER 3 Opt-in flow: DPA addendum + AdminPage panel + RollbackRecord V1 fix

Final third of the TIER 3 trilogy. Turns `Tenant.tier3_enabled` from a developer flag into a customer-facing legally-binding opt-in with a signed DPA addendum, downloadable PDF evidence, audit-grade revocation, and the `RollbackRecord.connection_id` write-time fix that closes Batch 22B's V1 limitation.

**Engineering decisions:**

- **DGE renderer NOT reused for the DPA PDF.** The Document Generation Engine expects a `ReportData` schema (financial-report shape — metrics, time_series, findings, attestation proof). The Tier 3 DPA is a free-form legal artefact, not a report. Forcing it through DGE would require lying about the schema. Instead used reportlab directly via the same brand-palette pattern as `scripts/build_dpa_pdf.py` (which renders the master DPA the same way). Rule #1 still respected — reportlab is the underlying primitive both modules use, no new dep, no new toolchain. Documented this deviation in `tier3_dpa/generator.py`'s module docstring so the next contributor sees the reasoning.

- **AdminPage.tsx extended, not rewritten.** Added `Tier3Section` + two modal components (`Tier3SignModal`, `Tier3RevokeModal`) alongside the existing `OverviewTiles` / `SystemStatusSection` / `UsersSection` / `ActivitySection`. The new section renders between Users and Activity. Three mutually-exclusive states (A platform off / B unsigned / C signed) match the backend status payload's flags one-to-one.

- **Two-click signing UX.** The legal weight of activating EXECUTE_CHANGE demands defence-in-depth on the UI side too — a single accidental click can't flip `tier3_enabled`. The section button opens a confirmation modal carrying the four-bullet summary (Fortnox-only, max 1 write/run, rollback available, No-Data still applies); only the modal's primary button posts to `/api/admin/tier3/sign-dpa`. Same pattern for revoke (with an optional free-text reason field).

- **`confirm=True` body field on sign endpoint.** Rejecting the request server-side without the explicit flag is rule #6 (Anthropic-grade safety, fail-closed). A misconfigured client that posts an empty body cannot accidentally sign the DPA.

- **Double-signing returns 409, not 400.** The state — "an active signature already exists" — is a conflict, not a malformed request. Operator sees a clean status code; the AdminPage doesn't need to special-case the message.

- **Denormalised state on Tenant + audit-grade truth in `tier3_dpa_signatures`.** The signature table carries the immutable record (with ip_address + user_agent for audit); the Tenant columns are the fast-read denormalisation for the admin panel. Both stay in sync via the same transaction. Revoke flips `Tenant.tier3_enabled=False` but preserves `tier3_dpa_signed_at` + version on the row as audit history.

- **`UNIQUE (tenant_id, dpa_version)` on signature table.** Same tenant can't double-sign the same version. A bumped version (v1.1 → v2.0) gets a fresh row with no conflict. Re-sign after revoke works because the constraint is on (tenant, version), not (tenant) alone.

- **`RollbackRecord.connection_id` is a first-class column now.** The Batch 22B V1 limitation was: the rollback handlers read `connection_id` from `prior_state` JSON, but the capabilities didn't put it there — they only stored the field they touched. Operators had to add it via the UI before a rollback would work. Batch 22C: `connection_id` is its own column. Capabilities pass it through `_persist_rollback_record(connection_id=...)`. Rollback dispatchers read `getattr(row, "connection_id", None) or (row.prior_state or {}).get("connection_id", "")` so historical pre-22C rows stay rollback-able (forward-compat).

- **Raw SQL `IF NOT EXISTS` migration.** Three schema changes in one Alembic revision (`10f575a96ce4`): the new table + index, two `ALTER TABLE … ADD COLUMN` on `tenants`, one on `rollback_records`. Replay-safe per the Batch 12 → 16 → 17 → 18 → 20 → 22B lesson.

- **No tests reused — built `tests/test_tier3_dpa.py` fresh.** Existing `test_tier3_executor.py` covers the EXECUTOR archetype + AVM tier-3 + Fortnox write contract; the DPA opt-in flow is orthogonal. 9 new tests:
  - 4 sign endpoint (creates row · enables tier3 · refuses confirm=False · refuses double-sign)
  - 2 revoke + status (revoke disables tier3 · status payload shape)
  - 1 PDF generation (`%PDF` header + non-trivial size + idempotent regen)
  - 2 gate + V1 fix (`tier3_enabled=False` blocks Fortnox writes · `RollbackRecord.connection_id` populated)

  Scaffolding mirrors `test_tier3_executor.py`: in-memory SQLite via StaticPool, `database.session.SessionLocal` monkey-patched, FastAPI TestClient with `get_db` + `get_current_user` overrides.

**Verification:**

- Backend pytest: **1202 passed** (1193 + 9, zero regressions, 183 s).
- Go suite (`golang:1.22-alpine`): all 7 packages green.
- Frontend: typecheck ✓, 60 tests ✓, Vite build ✓.
- Landing Next.js build: 33 static pages ✓.

**Manual pendings (unchanged from Batch 22B close-out):**

- Hetzner Postgres `alembic stamp head` still pending — production restart will fail until the operator runs the two commands documented in CLAUDE.md `## Production DB`. Head expected: `10f575a96ce4 (head)`.
- Branch-protection rule key needs to flip from `Backend — 1193 tests` to `Backend — 1202 tests` in the GitHub Settings UI (lockstep with the ci.yml `name:` already bumped here).
- `SESSION_MIRROR_TOKEN` still expired — the public mirror sync for `abn-session-memory` repo must be re-run manually after the push.

**What's next — Batch 23 (preview):**

Now that TIER 3 is end-to-end (foundation 22A · writes 22B · opt-in 22C), the natural next move is broadening the connector surface so TIER 3 isn't Fortnox-only:

- Quinyx write-back: schedule_change capability + AVM rules + Nango proxy
- Hogia write-back: invoice_status capability mirroring Fortnox's
- AVM industry-specific allowlists (`scheduling` industry pins Quinyx; `payroll` pins Hogia)
- DPA addendum versioning: bump to v1.1 when Quinyx/Hogia enter the allowlist; existing v1.0 signatures stay valid but prompt for re-sign on next AdminPage visit
- Frontend: AdminPage "Tier 3-aktiverade system" matrix replacing the single `fortnox_write_enabled` flag with per-connector toggles


## 2026-05-26 — Batch 22B — TIER 3 EXECUTE_CHANGE: EXECUTOR archetype + Fortnox write-back + RollbackRecord

Second half of the TIER 3 work. Batch 22A laid the cryptographic foundation (abn-security + canonical signatures); this batch wires the actual write-back. After this, ABN is no longer just an *Autonomous Observer Network* — it's the full *Autonomous Backoffice Network* the brand promises, gated behind a three-layer opt-in.

**Architecture decisions made:**

- **Two tier-3 switches, not one.** Platform-level `settings.enable_execute_change` (env-driven, ops-controlled) + per-tenant `Tenant.tier3_enabled` (DB-driven, legal-controlled). The DPA addendum is the platform-level legal opt-in; the tenant flag is the operational opt-in. **Both must be True.** Different stakeholders own different switches — operations can disable TIER 3 globally for any reason without touching tenant data; legal can disable per-tenant without touching ops config. Plus `fortnox_write_enabled` as the per-system kill-switch → three-layer defence.

- **AVM ALWAYS downgrades, never rejects.** Engineering rule #6 (fail-closed) interpreted via the human-in-the-loop principle: a misconfigured tier-3 blueprint shouldn't kill the run; it should surface as a tier-2 proposal. Downgrade chain: EXECUTOR → PLANNER, tier 3 → 2, `execute_change=False`, plus an `AVM_DOWNGRADE_TIER3: <reason>` warning so operators see what happened in the audit log. Four rules trigger downgrade: (a) regulated industry blocks EXECUTE_CHANGE, (b) EXECUTOR archetype without the explicit flag + step, (c) tier-3 blueprint missing signature, (d) EXECUTE_CHANGE step lacks `rollback_action`. None reject — all downgrade.

- **`paid` deliberately NOT in the allowed status set.** Spec said "Only allowed statuses: cancelled, reminded — never paid (financial record — too risky)". Honoured byte-for-byte: `ALLOWED_INVOICE_STATUSES = frozenset({"cancelled", "reminded"})`. Touching a financial record requires auditor sign-off + bookkeeping reconciliation; V1 stays at the safe boundary. The allowlist is the audit-friendly surface — adding `paid` later requires an explicit batch + a SOC2 readiness review.

- **Signature verified TWICE per tier-3 step.** OPERA Mission Layer pre-flight (Batch 22B) verifies once at the run boundary; each capability re-verifies before its Nango call. Two HTTP round-trips to abn-security — cheap; the safety margin is large. A Mission Layer bug can't bypass the gate because the per-capability check runs anyway.

- **`RollbackRecord` is the audit-and-undo seam, not a transactional log.** Written AFTER the Nango write succeeds — its existence is the proof the write actually happened. `prior_state` carries the field's value before the write (aggregate, never customer data — `{"status": "open"}` or `{"cost_center": "CC-100"}`). The rollback endpoint reads this and dispatches the reverse Nango call via `_ROLLBACK_DISPATCH`. Idempotent (second rollback call short-circuits on `rolled_back=True`).

- **`run_id` as String FK, not Integer.** Per the Batch-17+ convention — `AgentRun.run_id` is a UUID string, not the integer PK. Same shape as `Finding.run_id`, `ROILedger.run_id`, `NotificationDispatch`'s relation to runs.

- **5th ACTION_MAP category `execute` + `KIND_EXECUTE` constant.** The spec asked for an `execute.fortnox_*` action_map key shape. The existing four categories (fetch_data/analyze/propose/report) didn't accommodate EXECUTE_CHANGE actions, so a 5th category was the cleanest extension. Updated the existing `test_action_map_has_all_four_categories` → `_all_five_categories` so the test name reflects reality.

- **EXECUTOR archetype was already in `VALID_ARCHETYPES`** (since the initial Blueprint Generator). Batch 22B just wired `_derive_archetype_handbook` to actually produce it under the right conditions. New kwarg `tier3_enabled: bool = False` with backward-compat default — existing callers see no change in behaviour.

- **Reused `enable_execute_change` Settings field** rather than adding a new `tier3_enabled` Settings (which the spec hinted at). The existing field already covers the platform-wide concept (rule #2 — one secret/flag per env var). The tenant column is the per-tenant complement; the platform Settings is the platform-wide complement; they're not duplicates.

- **`_ROLLBACK_DISPATCH` lives in `admin.py`** — the rollback API endpoint is the inverse-action dispatcher, so the routing belongs next to the endpoint. Each handler re-uses the Fortnox helpers from `fortnox_write._fortnox_put_invoice_field` (rule #1 — extend the Nango plumbing, don't fork it).

- **V1 connection_id limitation documented.** The current rollback handlers require `connection_id` in `prior_state`, but the Batch-22B Fortnox writes don't store it there (only the field they touched). Real production rollback needs the connection_id baked into the row at write-time — that's flagged as Batch 22C work. The handlers raise a clean `RuntimeError("rollback requires connection_id in prior_state")` so the operator sees the V1 boundary instead of a silent failure.

**Reviewer caught (mid-batch):**

1. **Pre-existing `test_action_map_has_all_four_categories`** asserted exactly 4 categories. Adding `execute` broke it. Updated the test name + assertion to reflect the architectural change.

2. **`Tenant.tier3_enabled` would have collided with `Tenant.policy.max_tier`** if implemented naïvely as redundant config. Resolved by treating them as defence-in-depth: `tier3_enabled` is the platform-level boolean kill-switch, `policy.max_tier` is the granular tier ceiling. EXECUTOR archetype requires both (boolean True + max_tier == 3). They diverge intentionally for tenants who signed the DPA addendum but haven't yet operationalised tier 3.

**Test results:** 13 new tests in `tests/test_tier3_executor.py`. Full backend: **1193 passed** (1180 + 13), one pre-existing test name updated. Go: 7/7 packages green. Landing: 33/33 static pages.

**No-Data invariant audit (rule #6):**
- `RollbackRecord.prior_state` + `new_state` JSON columns hold only the field the capability touched — status string, cost-centre code, boolean flag. Never customer names, full amounts beyond the touched field, personnummer.
- `target_id` is the external Fortnox invoice number — an identifier the customer already has in Fortnox, not a value extracted from their data (same pattern as `Subscription.stripe_customer_id`).
- Rollback audit row writes `ABNActivityLog{activity_type="rollback", connector_type="fortnox", resource_name=action_type, sent_outside=0, pii_detected=0}` — counts only.
- All capability errors map to `FortnoxWriteBlockedError` with short reason strings — never log Nango response bodies.
- Three-layer opt-in check happens BEFORE any Nango call; even a misconfigured tenant can't trigger an outbound HTTP to Fortnox.

**End-to-end TIER 3 flow now functional:**
1. Tenant signs DPA addendum → operator flips `Tenant.tier3_enabled=True`
2. Operator sets `ENABLE_EXECUTE_CHANGE=true` + `FORTNOX_WRITE_ENABLED=true` in env
3. Blueprint Generator runs → detects `tier3_enabled_for_run=True` → produces tier-3 blueprint with `execute_change=true` + EXECUTOR archetype
4. AVM step 5b validates: tier-3 rules pass → blueprint signed via canonical signer (Batch 22A)
5. Agent run starts → OPERA `_tier3_preflight_or_raise` verifies signature cross-process via abn-security
6. Execute phase reaches `execute.fortnox_update_invoice_status` step
7. Capability does three-layer opt-in + signature re-verify + GET prior + PUT new + persist RollbackRecord
8. If something went wrong: NODE_ADMIN goes to `/admin/rollback-records` → clicks rollback → reverse Nango call restores prior state

Every step has its own fail-closed gate. The chain is end-to-end now; **closes the unbuilt half of the product promise** from the audit two days ago.

**Manual pendings (unchanged from Batch 22A close-out):**
1. `SESSION_MIRROR_TOKEN` still expired — manual mirror sync done.
2. Branch-protection rule key: `Backend — 1180 tests` → `Backend — 1193 tests`.
3. Hetzner DB `alembic stamp head` still pending since Batch 8 — now with one more migration on the chain (`b32c7e2ba5e5`).

## 2026-05-26 — Batch 22A — TIER 3 foundation: abn-security compiled + Go↔Python blueprint signatures

First half of the TIER 3 EXECUTE_CHANGE work. Two long-standing gaps closed in one batch:
1. `services/abn-security` (Go) compiles + tests **for the first time ever** since it was scaffolded in handbook ch. 11.
2. Blueprint signatures **round-trip across Python and Go** — same byte stream, same key, same digest.

**Architecture decisions made (all rule-driven deviations from spec):**

- **Module name `abnsecurity` (not `github.com/abn-systems/abn-security` as spec assumed).** Read existing `go.mod` first — module is `abnsecurity`, stdlib-only, no third-party deps. Used `abnsecurity/internal/blueprint` for the new package. Spec was guessing at the module path.

- **Env var unified to `ABN_NODE_SIGNING_KEY` (NOT spec's new `ABN_SIGNING_KEY`).** The existing abn-security `main.go` already reads `ABN_NODE_SIGNING_KEY` for the identity verifier. Adding a parallel `ABN_SIGNING_KEY` would create two confusing env vars for the same secret — exactly the kind of duplication rule #2 forbids. Reused the existing one. Python settings `node_signing_key` (which Docker Compose maps to `NODE_SIGNING_KEY` env var on the python container, then to `ABN_NODE_SIGNING_KEY` on the abn-security container) is the canonical name going forward.

- **`BlueprintVerifier` is a higher layer on top of existing `identity.HMACVerifier`** — not a fork of it. The HMAC primitive (constant-time compare over arbitrary strings) already lives in `identity`; the new package adds blueprint-specific logic (canonical JSON, metadata extraction, tier-3 flag enforcement). Two layers, single primitive (rule #1).

- **`api.Server` extended via `WithBlueprintVerifier` builder method, NOT rewritten.** The existing `NewServer(manager, engine, guard)` signature stays unchanged — existing tests + existing callers in `cmd/abn-security/main.go` see no behaviour change. The new `WithBlueprintVerifier(v, audit)` is opt-in builder style. When nil, the `/verify-blueprint` endpoint returns 503 fail-closed.

- **Two divergent Python signers existed before the batch started (REVIEWER CAUGHT):**
    - `agent_engine.blueprint_generator.sign_blueprint` used `json.dumps(payload, sort_keys=True, default=str)` — **default whitespace separators**, `, ` and `: ` (with spaces). Plus key `settings.blueprint_signing_key`.
    - I had to add a Go-compatible signer that uses `separators=(",", ":")` (no whitespace) and key `settings.node_signing_key`.
  - **Resolution**: New `backend/trust/blueprint_signer.py` is the SSoT for the canonical form. The legacy `generator.sign_blueprint`/`verify_blueprint` migrated to thin wrappers that delegate. Same function names, same import paths, same call sites — but now the canonical form is unified. Existing tests do round-trip (sign → verify in the same test) so they all still pass under the migration. The legacy `blueprint_signing_key` setting stays on Settings for backward compatibility but is no longer used by any new code; future batch can drop it cleanly.

- **Canonical form contract is now a single-source-of-truth Python module that the Go service mirrors byte-for-byte:**
    - `json.dumps(bp, sort_keys=True, separators=(",", ":"))` in Python.
    - `json.Marshal(map[string]any)` in Go — emits identical bytes for the same input because Go's stdlib JSON sorts top-level map keys and emits no whitespace by default.
    - **The contract is asserted in code**: `test_canonical_form_matches_go_reference` computes the Python signature by hand using `hmac.new(...).hexdigest()` over the byte stream and compares against `sign_blueprint(...)`. If anyone ever changes the canonical form in only one language, this test fails immediately. Go side has `TestSign_ProducesVerifiableSignature` which does the same round-trip on the Go side.

- **Tier-3 flag rule is defence-in-depth on top of HMAC.** A blueprint with `"tier": 3` must also carry a truthy `"execute_change"` field. Without it: `Valid=false` regardless of whether the signature is correct. Rationale: forged tier promotion (attacker takes a valid tier-2 signature and changes `tier` to 3 in a downstream system) gets rejected on TWO levels — first because changing `tier` invalidates the signature, second because the flag is missing. Belt-and-braces (rule #6).

- **Empty-key contract is FAIL-CLOSED.** `BlueprintVerifier` constructed with empty key responds to every Verify call with `Valid=false, Reason="signing key unconfigured"`. The HTTP endpoint short-circuits with **503 Service Unavailable** when `ABN_NODE_SIGNING_KEY` is empty. Defends against the "deploy with no env var" mistake — better to refuse all verifies than silently approve them.

- **Audit log uses short uppercase tokens, not free-form reason strings.** `audit.Logger.Log("blueprint", "verify", "DENIED_SIG_MISMATCH")` — grep-able, fixed vocabulary, never carries the actual signature or payload (No-Data invariant on the audit trail).

**Local Go compilation breakthrough:**
- No Go toolchain in the dev environment (known issue since handbook ch. 11 was first scaffolded).
- Tried `which go` → not installed. `winget install GoLang.Go` → download started but the path-resolution didn't surface a usable binary in this session.
- **Docker pivot**: ran `docker run --rm -v "/c/Users/Jacob/.../abn-security:/src" -w /src golang:1.22-alpine sh -c "go vet ./... && go build ./... && go test ./..."` — worked immediately. First-ever local compile of abn-security: **all 5 existing packages green** before any new code was added.
- After adding the new package + endpoint: **7/7 packages green** (75% coverage on `internal/blueprint`, 100% on `identity`, 88-95% on the others). The CI `security-go` job (which uses `actions/setup-go@v5`) runs the same Go 1.22 toolchain.
- This means **the abn-security service is now operationally trustworthy** for the first time. Every batch from here can include Go changes that get exercised locally.

**Test results:** 9 new Go tests + 9 new Python tests. Full backend: **1180 passed** (1171 + 9). All Go packages green. Landing 33/33. Zero regressions despite migrating the canonical signer.

**No-Data invariant audit (rule #6):**
- Canonical JSON form is computed on the dict EXCLUDING any raw fields the agent might have leaked there; we read keys like `agent_id`, `tenant_id`, `tier`, `signature`, `execute_change` — all metadata. The signed content is the blueprint structure (steps, capabilities, etc), all of which is aggregate-by-construction.
- HTTP request/response payloads carry base64-of-JSON in transit but never log it.
- Audit entries carry only `{system, subject, decision}` — short tokens, never the blueprint itself.
- Failure reasons are mapped to short uppercase codes (`SIG_MISMATCH`, `TIER3_NO_FLAG`, etc) — no free-form text in the audit log.

**Manual pendings (unchanged from Batch 21 close-out):**
1. `SESSION_MIRROR_TOKEN` still expired — manual mirror sync done for this batch.
2. Branch-protection rule key: `Backend — 1171 tests` → `Backend — 1180 tests`.
3. Hetzner DB `alembic stamp head` still pending since Batch 8.

**What's next — Batch 22B (the second half of TIER 3 EXECUTE_CHANGE):**
- `EXECUTOR` archetype in `blueprint_generator._derive_archetype_handbook` (today returns one of MONITOR/ANALYZER/PLANNER/COORDINATOR; needs the fifth).
- AVM rules for tier 3 — forbidden by default for every industry; opt-in per-tenant flag.
- Per-connector write capabilities — first `Fortnox.create_invoice` + `Fortnox.update_invoice` (highest-impact for SE customers).
- OPERA Mission Layer pre-flight via `verify_blueprint_via_security_service` — refuses to start any tier-3 step on `Valid=false`.
- Write-back rollback semantics in the OPERA Act phase.

## 2026-05-26 — Batch 21 — Wire Proposal creation into OPERA runtime (Tier-2 end-to-end)
Closes the gap I flagged in the "Top-3 most impactful missing things" answer earlier today: every Tier-2 surface ABN has built since Batch 11 was infrastructurally complete but functionally empty, because no OPERA path created Proposal rows. Batch 21 lands the bridge.

**Architecture decisions made (deviations from the spec, all rule-driven):**
- **Helper calls `evaluate_commitment` internally — does NOT mutate `OPERARunResult`.** The spec's TASK 4 said "Add `commitment_decision` field to `OPERARunResult`". Avoided because threading a stable dataclass field through the runner for a value the helper can compute itself couples two layers for no benefit. The gate is pure logic — same input (output + confidence + blueprint), same output. Cleaner: OPERARunner stays unaware of the gate; the persistence seam owns the mapping.
- **Helper lives in `agent_runtime/`, not `agent_engine/`.** The spec said `backend/agent_engine/proposals_persistence.py`. I moved it to `backend/agent_runtime/proposals_persistence.py` because: `commitment_gate` lives in `agent_runtime/execution/`; `findings_persistence` lives in `agent_runtime/`; the OPERA runner already imports both from `agent_runtime`. Same module = same import discipline. Rule #1 — extend the existing structure, don't fork a new one.
- **NO redundant `deliver_proposal_notification` call in `_save_run_record`.** The spec's TASK 2 second half ("call deliver_proposal_notification on the newly-created Proposal") would have duplicated the Batch-18 block immediately downstream. That block was specifically designed to pick up any Proposal row written during the run via a `Proposal.run_id == self.run_id` query — it's the consumer side waiting for exactly this producer. Adding a direct call would double-dispatch. Rule #1, hard line.
- **TASK 3 (draft_email wiring from commitment_gate) deliberately skipped.** Audit confirmed `CommitmentDecision.action ∈ {auto_deliver, deliver_with_flag, save_proposal, escalate}` — there is no `draft_email` action. Batch 20's `email.draft_email` capability is invoked directly via the `propose.draft_email` ACTION_MAP entry; the commitment gate runs alongside it and decides `save_proposal` on the same run if tier-2. Net result: an agent that drafts an email + the run lands a Proposal row with `action_type="report_delivery"` (per the `_PROPOSE_ACTION_MAP` mapping) — the human approves both at once on the dashboard. No new wiring needed for the email case; it falls out of the proposal mapping naturally.
- **Field names diverged from the spec to match the real DB schema.** The spec asked for `Proposal.confidence` and `Proposal.proposal_payload`. Audit showed those columns don't exist — `Proposal` has `output_summary` (JSON) and confidence lives on `AgentRun`. Used the real columns (rule #2 — never invent fields). Confidence is preserved inside `output_summary.decision.confidence` for the audit trail.
- **`run_id: str`, not int.** The spec wrote `run_id: int` but `Proposal.run_id` is `Column(String, ForeignKey("agent_runs.run_id"))` — the UUID string, not the integer PK. Used the string consistently.
- **Action-type mapping uses the existing `VALID_PROPOSAL_TYPES` set from `api.routes.proposals.py`.** Five values total: `dispute_letter`, `schedule_change`, `threshold_adjustment`, `report_delivery`, `agent_pause`. Unknown `propose.*` actions fall through to `agent_pause` (safest default: proposal still surfaces, operator pauses the agent for review). Tested by `test_action_type_derivation_*` patterns.
- **Idempotency identical to `save_findings_from_output`** — pre-existing `(run_id, tenant_id)` row returns its id; no double-write. The proposals helper diverges from findings in one detail: findings deletes-then-inserts for the multi-row case; proposals is one-row-per-run so it returns the existing id without re-inserting.
- **`impact_summary` is aggregate-only Swedish** — `"3 fynd · potentiell besparing EUR 18 600"`. Never raw values. Built from `len(findings)` + `sum(findings.impact_eur)` — both already aggregates by definition. NBSP thousands grouping (Swedish convention).
- **`output_summary` JSON captures the decision audit trail** — `{action, tier_used, original_tier, downgraded, confidence, reason}` from `CommitmentDecision.to_dict()` plus the run's `summary`, top-5 `recommendations`, top-10 `report_paths`. Every field is aggregate or LLM-narrative; no raw values.

**Test results:** 10 new tests in `tests/test_proposal_wiring.py`. Full backend: **1171 passed** (1161 + 10), no regressions. Landing 33/33. Frontend typecheck clean (this batch is backend-only).

**No-Data invariant audit (rule #6):**
- `impact_summary` is aggregate-only (count + total).
- `output_summary.recommendations` and `output_summary.summary` come from the OPERA Reason phase, which already enforces No-Data via the LLM Gateway's redact-mode policy. The helper truncates at 500/200 chars defensively.
- `output_summary.report_paths` are filesystem paths — never raw customer values (per the existing OPERA report-generation contract).
- The commitment gate's `reason` is a one-liner about confidence + tier — never carries customer data.
- `action_type` and `target_system` are enum-bounded short strings.

**End-to-end audit verifying Tier-2 actually works now:**
1. Agent run starts (tier=2 blueprint), executes OPERA, produces output with confidence 0.82
2. `_save_run_record` runs: AgentRun + Findings + ROI + Semantic + AgentMemory writes (all unchanged)
3. **NEW Batch-21 block** calls `save_proposal_from_run` → gate decides `save_proposal` → row inserted with `status="pending"`, `action_type="schedule_change"`, `target_system="quinyx"`, `impact_eur=18600`, `expires_at=now+14d`
4. **Existing Batch-18 block** queries `Proposal.run_id == self.run_id, status IN (pending, pending_approval)` → finds the new row → resolves `TenantDeliveryConfig` → calls `deliver_proposal_notification` → Slack/Teams Block-Kit/Adaptive-Card with Godkänn/Avvisa buttons appears in the customer's channel
5. Human clicks ✅ → `POST /api/slack/interactions` signature-verifies → calls `proposals._record_decision` → flips `Proposal.status = "approved"` + writes `ApprovalRecord`
6. Dashboard `/proposals` shows the new row from step 3 (status: pending) or step 5 (status: approved)

Every link in that chain existed individually before Batch 21; **the missing piece was step 3**. Now closed.

**Manual pendings unchanged from Batch 20 close-out:**
1. `SESSION_MIRROR_TOKEN` still expired — auto-sync workflow keeps failing; manual mirror sync for Batch 21 same pattern as Batches 19/20.
2. Branch-protection rule key: `Backend — 1161 tests` → `Backend — 1171 tests`.
3. Hetzner DB `alembic stamp head` still pending since Batch 8.

## 2026-05-26 — Batch 20 — Email capability (read inbox → draft → human approves → never auto-send)
Jacob requested the full agents-handle-mail surface: read inbox metadata, draft outbound, get human approval, send via existing deliverers. The audit two turns earlier laid out exactly what was missing.

**Architecture decisions made:**
- **Agents NEVER auto-send.** Two layers of defence: (a) the API approve endpoint is the only code path that calls `send_approved_draft`, gated on `require_role(NODE_ADMIN|AGENT_MANAGER)` + a status check; (b) `send_approved_draft` itself refuses to run unless `EmailDraft.status == "approved"`. A blueprint that lists `report.send_approved_email` in its action chain is a no-op until a human clicks Godkänn.
- **Inbound = metadata only — subject/body BLOCKED at the normaliser layer**, not just "filtered in the API". The normalisers' `FIELD_MAP` is a whitelist; anything not in it never reaches the events table in the first place. PII Guardian runs as a defensive second pass on survivors. The dataclass + the events table physically cannot carry mail content.
- **`from_domain` not `from_address`.** `"Alice Andersson <alice@acme.se>"` → `"acme.se"`. Agents can correlate counterparty *patterns* (this domain talks to us X times a week, X% have attachments) without ever exposing who. The local part is dropped at normaliser-time, not at API-time — so even an internal log query can't recover it.
- **Recipients stored as SHA-256 fingerprints, never raw addresses.** Cleartext addresses pass through the approve handler at send-time (the operator already knows them under their RBAC) and are cross-checked against the stored fingerprints; mismatch refuses the send. The fingerprint is the tamper-evidence cross-check, not an obfuscation: if a malicious actor edits the DB row to point at a different recipient, the next approve attempt fails because the operator's supplied list won't hash to the same set.
- **Template validation rejects raw values at draft time.** `_validate_template_no_raw_values` heuristic: any 4+ digit run (amount / invoice number / personnummer pattern), any literal `@domain` string (recipient should already be a fingerprint), 2 KB body length cap (template, not content). Engineering rule #6 fail-closed: when in doubt, raise — real templates use placeholders like `{{count}}` / `{{amount_band}}`.
- **`ApprovalRecord` reused for the draft audit** (rule #2 — single source of truth for human approvals). The `proposal_id` column carries the `"draft:{id}"` namespace so a single SELECT on `approval_records` covers both proposal approvals and draft approvals. The audit page can render both flows from one table.
- **YAML `direction: both`** for `gmail.yaml` + `outlook.yaml` — these are now legitimately bidirectional (read metadata + send approved drafts via the same Nango integration). One pre-existing test asserted Gmail was outbound-only; updated to assert `direction: both` since that's now correct.
- **Capability `(context, adapters) -> context` signature preserved** (rule #1 — every other capability uses this shape). The kwargs come in via `context["draft_email"]` / `context["read_inbox_filters"]` so the existing `run_capability(name, context, adapters)` call site doesn't need to thread additional `**kwargs` through.
- **`SessionLocal()` inside the capability**, not threaded from the runner. The OPERA runner's session is the run-scope session; the draft writer needs a self-contained transaction (the draft outlives the run). Same pattern as the LLM call log writer from Batch 13B.
- **Three deliverers stay unchanged — `send_approved_draft` builds an `AgentReport` payload and calls `deliver_report`.** Rule #1: the send mechanics already exist; the email capability just supplies the payload. No new deliverer methods. SendGrid's `_strip_numbers` doesn't break drafts because draft templates already don't carry raw values (validated at draft time).
- **`asyncio.run` from sync capability with a fallback for nested-loop hosts** — `send_approved_draft` runs synchronously (matches capability signature) but the deliverer methods are async. The function falls back to `loop.run_until_complete` when invoked from inside a running loop (Tauri sidecar, AAEA executor).
- **`@router` mounts paths individually, not under a single prefix** — `/api/agents/{id}/drafts` lives under `/api/agents`, `/api/drafts/*` under `/api/drafts`. Both surfaces in the same router file means one import in main.py.

**Reviewer caught:**
- Initial test assertions used a top-level `from database.session import SessionLocal` which captured the production sessionmaker — even after `_override_sessionlocal` monkeypatched `database.session.SessionLocal`, the test's reference was stale. The capability's own late-import `from database.session import SessionLocal` inside the function body picked up the monkeypatch correctly (re-imported every call). Fix: test assertions use the test `factory()` for queries; the capability's writes go to the correct test engine via the monkeypatch.
- The `_fake_send` monkeypatch needed to use the closed-over test `factory` for its DB writes, not the top-level `SessionLocal`. Same root cause as above.
- Pre-existing `test_catalogue_endpoint_exposes_inbound_and_outbound_directions` asserted `gmail.direction == "outbound"`; updated to `"both"` since the gmail.yaml extension legitimately makes it bidirectional, and added a positive `slack_delivery.direction == "outbound"` assertion to keep the test's intent (verify direction classification works for the three cases: inbound-only, outbound-only, both).

**Test results:** 13 new tests in `tests/test_email_capability.py`. Full backend: **1161 passed** (1148 + 13), one pre-existing test updated to reflect the YAML change. Frontend: 60 vitest + Vite build + typecheck clean. Landing: 33/33 static pages.

**No-Data invariant audit (rule #6):**
- Subject + body never reach the events table — blocked at normaliser FIELD_MAP whitelist. Verified by `test_gmail_normaliser_blocks_subject` + `_blocks_body` + `test_outlook_normaliser_blocks_subject`.
- Full From/To/CC/BCC addresses never reach the events table — only `from_domain` survives.
- Recipient addresses never reach the email_drafts table — only SHA-256 fingerprints. Verified by `test_draft_email_stores_fingerprint_not_address`.
- Template strings rejected if they embed raw values (digits, literal @domain). Verified by `test_draft_email_never_stores_raw_values`.
- API never returns fingerprints — only counts. Verified by `test_drafts_api_returns_recipient_count_not_fingerprints` (asserts no SHA-256 hex pattern in response body).
- Cleartext recipients live only inside the approve-handler's runtime context for the duration of a send; never persisted. The fingerprint cross-check refuses a mismatched supply.
- Capability fail-closed: `send_approved_draft` refuses on missing draft_id / draft not approved / unknown channel. Verified by `test_send_approved_draft_requires_approval` + `_rejects_pending`.

## 2026-05-26 — Batch 19 — Admin Dashboard + ABN Node landing copy
Jacob requested the admin surface + a landing-copy refresh ("ABN" → "ABN Node" on the download page). The audit revealed that the Admin Dashboard is actually two very different surfaces; only one of them fits ABN's architecture.

**Architecture decisions made:**
- **Surface A only — per-Node admin.** The NODE_ADMIN's view of their own Node. Every endpoint scoped to `current.tenant_id`. No cross-tenant queries ever.
- **Surface B (cross-tenant fleet admin) audited + explicitly rejected.** Populating it requires either (a) pulling per-Node aggregates back to a central ABN store — at which point "no customer data leaves the Node" stops being architecturally enforceable, or (b) an aggregate-only opt-in telemetry channel which is its own batch with its own DPA addendum. The information ABN Systems AB actually needs for fleet ops already exists outside the Node: revenue via Stripe Dashboard, downloads via GitHub Releases analytics, uptime via external monitoring of `api.abnplatform.com`.
- **Aggregate existing tables, never new persistence.** `/api/admin/overview` is one query per existing model (Tenant, Agent, AgentRun, Finding, ROILedger, Proposal, User, Subscription, Connector) — no new tables, no new audit columns. Engineering rule #2 (one source of truth) applies in reverse: the existing tables already hold every fact the admin tile shows.
- **Reuse `require_role(Role.NODE_ADMIN)`, never re-implement RBAC.** All three admin endpoints declare the existing FastAPI dependency factory. Rule #1 in spirit: the auth scaffold from Batch 1 was built for exactly this surface.
- **Surface `MAX_NODE_ADMINS_PER_ORG` constant via the API, not hard-coded in the frontend.** The cap (3) lives in `auth.rbac`; the overview payload carries it so a future cap change touches one file.
- **Sidebar link visible to everyone, page itself is the gate.** Same pattern as `Billing` — non-NODE_ADMIN users see the link but `/admin` renders an "Ingen behörighet" card when the backend returns 403. Threading `currentUser.role` through the layout to hide the link would add state for cosmetics; the access control is at the API layer where it belongs.
- **`AdminPage.tsx` mirrors `BillingPage.tsx` exactly** — same TanStack Query 30s refetch, same `Card` / `PageHeading` / `StatusBadge` / `Table` from `components/ui.tsx`, same Swedish copy style, same Stripe-period date rendering. Rule #1 — pattern reuse, not a new design language.
- **The activity-log "Sänt utanför" column renders `0 byte` in green.** Carries the same No-Data signal the audit page uses — the operator sees at a glance that ABN's outbound traffic for that activity was zero, every row.
- **Three scheduler singletons (`observer_scheduler`, `agent_scheduler`, `friday_report_scheduler`) already export `.status()`** (Batches 9, 16) — `/api/admin/schedulers` is a thin pass-through. Each returns `{"available": True, ...payload}` on success, `{"available": False, "error": ...}` when the import fails — the frontend can render the same UI either way.
- **Landing copy: minimal, surgical changes only.** Two edits in `landing/app/company/download/`: the `<h1>` + the page metadata `title`, plus a new explainer `<section>` above the download accordion. No restructuring, no layout changes — the existing dark-theme styling (border `#2A2040`, card `#111118`) extends to the new block.
- **The architectural noun "Node" was already canonical in code** (`NODE_ADMIN` role, `node_id` setting, `node_signing_key`). Batch 19 just makes the customer-facing copy match.

**Reviewer caught:**
- Initial test seed for `Finding` rows failed with `NOT NULL constraint failed: findings.id` because `Finding.id` is a UUID **string** (not autoincremented Integer like the rest of the schema). Fixed by setting explicit `id=f"f-{i}"` per row.

**Test results:** 9 new tests in `tests/test_admin_routes.py` covering auth (requires_node_admin × 2, non_admin_gets_403, unauthenticated → 401/403/422), shape (full payload contract), correctness (agent counts; 7-day window filters out runs from 30 days ago + correctly counts in-window findings + correctly sums ROI; both `pending` and `pending_approval` count as pending), activity-log filtering (days=7 surfaces 2 rows, days=60 surfaces all 3, unknown activity_type filter → 0), schedulers (returns `available` flag per singleton).

**Verification:**
- Backend: **1148 passed** (1139 + 9), no regressions
- Frontend: 60 tests passed + Vite build OK + typecheck clean
- Landing: 33/33 static pages compiled

**No-Data invariant audit (rule #6):**
- Every count in the overview is filtered by `current.tenant_id` — admin can only see their own tenant's rows. RBAC scope = data scope.
- Activity log returns aggregate columns only (`records_read` / `pii_detected` / `sent_outside`) — never per-record customer values; the underlying `ABNActivityLog` rows don't carry them in the first place.
- The frontend never embeds the activity_id or any UUID in URLs or screenshots-friendly text; everything visible is type / status / counts.
- The Node identity (`node_id`, `env`) is surfaced — these are infrastructure identifiers, not customer data.

## 2026-05-26 — Batch 18 — Teams/Slack notifications + approval buttons
Jacob requested the full approval-button subsystem: outbound buttons in Slack + Teams, signature-verified webhook receivers, HMAC defence-in-depth, OPERA wired, 14 tests.

**Architecture decisions made:**
- **Extend, never rewrite (rule #1).** `slack_deliverer.py` and `teams_deliverer.py` kept all existing methods (`deliver_report`, `deliver_alert`, `test_connection`) byte-for-byte intact. `deliver_proposal` was added as a new method on each class. The Slack deliverer's existing `deliver_report` still uses legacy `attachments[]` (existing customers' reports unchanged); only the new `deliver_proposal` uses modern Block Kit `blocks[]`.
- **`ProposalNotification` dataclass defined once in `slack_deliverer.py`, imported by `teams_deliverer.py`.** Single source of truth for the payload shape (rule #2). No raw values — only aggregate fields (impact_eur, change_type, agent_name).
- **`_record_decision` is THE approval contract.** The Slack + Teams webhook receivers call the existing `api.routes.proposals._record_decision` and never re-implement the lifecycle / ApprovalRecord write. They are bridges, not parallel implementations.
- **Always return 200 OK.** Even on signature failure, replay rejection, malformed payload. Both Slack and Teams aggressively retry failed webhooks; returning a 4xx/5xx creates a feedback loop where bad payloads hammer the endpoint. The decision is visible via the `abn.api.{slack,teams}_interactions` logger and via `NotificationDispatch.status` column.
- **Two-layer security: webhook signature + dispatch_token HMAC.** The webhook signature (Slack v0:ts:body / Teams bearer) protects against unauthenticated senders. The `dispatch_token` (HMAC-SHA256 over `proposal_id`) protects against a *valid* sender submitting the wrong proposal_id with a stale token. Both must hold. Defence-in-depth, not redundancy — they catch different attack classes.
- **5-minute replay window.** Slack's documented value; rejects requests with `|now - timestamp| > 300s`. Stops replay attacks where an attacker captures one valid webhook and re-submits it.
- **Constant-time compares via `hmac.compare_digest`.** Three places: Slack signature verify, Teams bearer-token verify, dispatch_token verify. Prevents timing side-channels.
- **NotificationDispatch.status is the idempotency anchor.** A second click on the same message sees `status != "pending"` and short-circuits. Critical because:
  (a) Real users sometimes double-click before the original message is updated.
  (b) Slack retries `POST /api/slack/interactions` on transient failures.
  (c) The `unique(channel, external_message_id)` index in the DB is the last line of defence.
- **Fail-closed on missing secrets.** All three settings default to empty string. The webhook receivers refuse to process when `signing_secret` / `validation_token` is empty — a fresh install can never accidentally accept an unverified interaction. `DeliveryRouter.deliver_proposal_notification` refuses to dispatch when `notification_signature_secret` is empty — a forge-able token is worse than no token.
- **Bearer-token for Teams V1, JWT-via-JWKS as documented V2 path.** The spec says JWT validation but flags "simpler for V1". Going with the bearer token now lets us ship; the upgrade path is documented in CLAUDE.md `## Teams/Slack Approvals`. The bearer token is production-grade when paired with a strong shared secret (no different in security model from Stripe's webhook secret).
- **`compute_dispatch_token` lives in `delivery/router.py`, not a new module.** Used by both directions — the router computes when dispatching, the webhook receivers verify when receiving. Co-locating with the router avoids a circular import and keeps the function near its primary caller.
- **OPERA hook fires from `_save_run_record`, queries Proposals by `run_id`.** Today no production OPERA path creates Proposal rows (AAEA's commitment gate emits the decision but the actual insert is a later batch). The hook is here so when proposals start being created from this phase, notifications fire automatically. Fail-silent — a dispatch failure never blocks the run.
- **Channel preference: Slack first, Teams fallback.** Slack tends to be where ops teams already live; Teams is the Microsoft-shop alternative. SendGrid (the email fallback for reports) is NOT a destination for proposals — proposals need synchronous button clicks, not email-and-pray.

**Reviewer caught:**
- Initial T4 code called `tenant_config.get_channel_metadata(...)` which doesn't exist. Fixed by using the codebase's canonical `_resolve_active_with_connection_ids` helper (rule #2 — one source of truth for the connection-id lookup) and reading channel destinations directly from `TenantDeliveryConfig.slack_channel_id` / `.teams_channel_id` / `.teams_team_id`.
- Initial tests held a SQLAlchemy row across a session close → `DetachedInstanceError` when re-accessing attributes inside the test. Fixed by capturing `proposal.id` + `proposal.proposal_id` as plain values *before* the seed session closes, then re-querying via the test-bound Session factory inside each test. Pattern carried forward from Batch 17.

**Test results:** 14 new tests in `tests/test_slack_teams_notifications.py`. Full backend: **1139 passed** (1125 + 14), no regressions. Landing build: 33/33 static pages.

**No-Data invariant audit (rule #6):**
- `ProposalNotification` carries only aggregate fields — agent_name (human-readable), change_type (enum), impact_eur (RAL-attested aggregate), impact_summary (short customer-facing string, already on the Proposal row).
- The Block Kit / Adaptive Card body shows the EUR figure (deliberately — customers need it to make the call); never raw invoice numbers, vendor names, or amounts beyond the aggregate.
- The button `value` / `data` carries only `(action, proposal_id, dispatch_token)` — no payload data passes through the channel.
- Webhook receivers never log the body content beyond status codes + proposal_id; logs are aggregate metadata only.
- `NotificationDispatch.signature_token` is the HMAC, not the secret — exposing the row's `signature_token` value via a hypothetical leak doesn't compromise other proposals (each carries a different proposal_id input to the HMAC).

## 2026-05-26 — Batch 17 — Agent Memory (three layers)
Jacob requested the full Agent Memory subsystem: persisted EvolutionState + 7-day short-term + never-deleted long-term, OPERA wired, API surface, 14 tests.

**Architecture decisions made:**
- **Three tables in one Alembic migration.** The spec asked for T1 to be a separate revision from T2+T3. Autogen lumped all three (since they were added to models together), and splitting them adds zero operational value — the three tables are one cohesive subsystem, introduced together. The hard constraint that actually matters (raw SQL `CREATE INDEX IF NOT EXISTS` for replay safety, Batch 16 lesson) is preserved. Migration `7ab8394c998a` covers all three.
- **`agent_id: Integer` not `String`.** The spec's pseudo-code wrote `agent_id: str`, but the canonical FK pattern everywhere in the schema (Finding, AgentRun, ROILedger, AgentSettings, AgentActivityLog) is `Integer`. Followed Engineering rule #2 (one source of truth for the FK type) and used Integer. `tenant_id` stays String (consistent with the rest of the schema).
- **`Column(String, ...)` not `Column(DateTime, ...)`.** The spec said "DateTime NOT NULL default utcnow". Every other table in `models.py` stores datetimes as ISO strings — kept the convention.
- **`ShortTermMemory.no_raw_values_check` is the single source of truth for the No-Data policy** — re-used by `LongTermMemory.write` (rule #2). Heuristic: any string > 50 chars rejects as "looks like a raw value, not an aggregate". Engineering rule #6 (fail-closed): rejects on the borderline, never accepts on the borderline.
- **Impact stored as a coarse band (`small`/`medium`/`large`/`huge`), not raw EUR.** New helper `opera.runner._impact_band` does the mapping. Even though `ROILedger.impact_eur` already lives in the DB (RAL-attested), letting it leak into memory would defeat the No-Data invariant — memory is consumed by the Plan phase, which could feed it back to the LLM. Bands are aggregate-by-construction.
- **Confidence: max wins on reinforcement.** When the same fingerprint is re-observed, `confidence = max(prior, new)`. Reinforcement should never lower the bar. If a finding came in once at 0.9 and again at 0.6, the agent's belief stays at 0.9 — the second observation isn't evidence the first was wrong.
- **Promotion does NOT delete short-term.** Spec: "does NOT delete short_term (short_term expires naturally)". The short-term row stays useful for the freshness signal it carries even after promotion. Long-term carries durability; short-term carries recency.
- **Decay only multiplies confidence by 0.8, never deletes.** Rationale: a fact the agent hasn't seen in 6 months might still be true (a vendor going quiet for a quarter doesn't invalidate everything ABN learned about them). Better to surface low-confidence than to forget. Floor at 0.0 — never below.
- **API exposes `memory_type`/`fact_type`/`confidence`/`times_reinforced` only. NEVER `fact_key` or `fact_payload`.** Even though `fact_key` is a SHA-256 fingerprint (already No-Data-safe), exposing it gives a relying party a stable correlation handle — "is this the same counterparty across two API calls?" — which turns internal aggregates into queryable metadata. Customers asking "what has this agent learned?" get types + counts + confidence; deeper questions go through transparency.
- **Read-through TTL cache, not write-through.** `load_state` returns the cached row when fresh, recomputes + upserts when stale. The OPERA Plan phase always calls `load_state` (never `compute_state` directly), so a hot cache is essentially free; a cold cache pays one compute + one upsert. Cache TTL is per-stage (`cache_ttl_hours` on the EvolutionState row — 1 h Bootstrapping → 12 h Mature). 
- **Memory is reporting, not critical path.** Every read + write in the OPERA wire-up has its own try/except. A memory failure logs a warning and returns safe-empty defaults; the agent continues with broader thresholds. Engineering rule #6 fail-closed semantics: when the cache is unavailable, behave as if the agent is at Bootstrapping (broad thresholds, all alerts shown).
- **`Agent.created_at` as the agent's `first_run_at`.** The OPERA wire-up reads `Agent.created_at` to bootstrap `EvolutionState.days_active` on cache miss. Reasonable proxy — the agent's age in days drives stage selection, and a freshly-generated blueprint has no AgentRun rows to lean on.

**Test results:** 14 new tests in `tests/test_agent_memory.py` covering evolution persistence + TTL respect, short-term write/read/reinforcement/expiry/policy-rejection, long-term write/read/promotion/decay/never-deletes, OPERA injection + failure-is-silent, and the API privacy contract (assertions that neither `fact_payload` nor `fact_key` ever appear in the response body). Full backend: **1125 passed** (1111 + 14), no regressions. Landing build: 33/33 static pages.

**No-Data invariant audit (rule #6):**
- ROI Ledger writes `impact_eur` directly (RAL-attested) — but memory reads it AS a band via `_impact_band`. The raw EUR figure never reaches memory.
- Short-term `key` is `compute_fingerprint(...)` from `trust/fingerprint.py` — SHA-256 over agent_id + source_system + field_name + title prefix (max 48 chars). Title prefix is the finding's headline; the headline IS shown in the dashboard, but it's already the customer-facing summary (not the raw value). Per the existing pattern in `Finding.title`, this is the No-Data-safe surface.
- `value_json` carries `{"count": 1, "impact_band": "...", "severity": "..."}` — all aggregates.
- Long-term promotion preserves the same payload shape — no raw values introduced.
- API response is metadata-only: types, counts, confidences, dates. No keys, no payloads.

## 2026-05-26 — Batch 16 — Friday Report viral-loop e-mail
Jacob requested the full Friday Report subsystem: aggregate weekly value (EUR saved, hours saved, agents active, runs completed), branded HTML e-mail, cron Fri 07:00 Stockholm, three API routes, idempotency per `(tenant_id, week_ending)`, 10+ tests.

**Architecture decisions made:**
- **Separate `friday_report/sender.py` instead of reusing `delivery/sendgrid_deliverer.py`.** SendGridDeliverer's `_strip_numbers()` defensive scrub rewrites every digit in the body to `[…]` — correct for agent-output e-mails (any number could be a raw customer value) but catastrophic for the Friday digest, whose *entire point* is aggregate numbers (EUR, hours, run counts). These ARE the No-Data-safe figures the system is allowed to show — they come from RAL-attested ROILedger rows already filtered through the No-Data pipeline. So the digest builds its own SendGrid call with the same endpoint + auth, but without the scrub.
- **Three-layer idempotency** instead of one: the cron grace window (only one tick per week returns True from `_is_friday_morning`); a runtime `_already_sent_this_week` guard that checks for an `email_status="sent"` row; the model's unique `(tenant_id, week_ending)` index as the database-level last line of defence. Failed/skipped rows do NOT block retries — that's the whole point of recording status separately from existence.
- **Marketing-grade `hours_saved = total_eur / 50.0`** — 50 EUR/h is deliberately conservative for the Swedish labour market (mean professional gross labour cost ≈ 70 EUR/h). Constant is module-level so customer success can tune without code change. Honest enough for a weekly digest; the dashboard's per-finding view stays the auditable surface.
- **Empty-tenant path: log a `skipped` row, don't send.** A quiet week is a signal Customer Success uses to spot inactive tenants — but a digest titled "ABN did nothing for you" would be the wrong viral-loop artefact. Compromise: scheduler invokes `send_friday_report(..., dry_run=True)` so the audit row exists, the e-mail doesn't.
- **`report_date` query param** on `/preview` + `/send` body. Lets tests target a specific Friday without time-travelling the clock; also genuinely useful for the dashboard's "preview last week's digest" flow. Tolerates both date-only and full ISO timestamps.
- **Naive Stockholm clock with `zoneinfo` fallback to UTC+1h.** Hosts without tzdata (minimal Alpine) get a clock that's wrong by up to an hour during DST; the ±5 min grace window absorbs this. Right trade-off for a daemon that fires once a week.

**Critical bug discovery + fix (out-of-scope hotfix, landed in same batch):**
- The Batch 12 migration `8f8acda9625d_align_schema_drift.py` used `op.create_index(..., postgresql_if_not_exists=True)` on 7 separate index creations. SQLAlchemy 2.0 raises `ArgumentError: Argument 'postgresql_if_not_exists' is not accepted by dialect 'postgresql' on behalf of <class 'sqlalchemy.sql.schema.Index'>` — that kwarg is only honoured on `Table.create()`, NOT on `Index()`. The original Batch 12 hotfix never crashed in dev because the local Postgres was already at head when I "tested" it — the buggy code path never ran. The new Batch 16 migration ran first, hit the same error, and surfaced the latent bug in the prior migration.
- Rewrote BOTH migrations to use raw SQL `op.execute("CREATE [UNIQUE] INDEX IF NOT EXISTS ...")` — same end state, but actually works on a virgin DB. Per Engineering rule #6 (fail-closed, Anthropic-grade safety): fixing the prior migration belongs in this batch because Batch 16's deployment on Hetzner would crash *before* its own migration ever runs.

**Test results:** 12 new tests in `tests/test_friday_report.py` covering aggregation correctness (closed week-window math, out-of-week exclusion, empty-tenant zeros), deterministic e-mail render (byte-equality two renders, brand colours, Swedish copy, no raw values), idempotent sends (skip-no-recipients, dry-run, failed-then-replaced, send-twice-yields-one-row), scheduler already-sent guard, and the 3 API routes (preview returns aggregates+HTML, send is idempotent through the API, history orders + limits). Full backend: **1111 passed** (1099 + 12).

**No-Data invariant audit (rule #6):**
- ROILedger rows are written by `agent_runtime.roi_ledger.writer.write_roi_entry` AFTER the LLM gateway has scrubbed every value — they only contain `impact_eur` (a derived aggregate) + `agent_id` + `run_id`. No raw customer values touch the Friday Report path.
- `top_agents` rollup uses `Agent.agent_name` — the human-readable agent name set by the Blueprint Generator (e.g. "Fraktfakturarevisor"). No customer-identifiable strings.
- Recipients live in `TenantSettings.notification_emails` — these are the customer's own staff e-mail addresses, deliberately set by them.
- The e-mail body contains the customer's tenant name (deliberately, so the digest is personalised), aggregate counts, and the EUR/hours totals. No invoice numbers, no employee names, no GPS coordinates, no PII.
- SendGrid send goes from `reports@abnplatform.com` to addresses the customer chose; never proxies via Nango (no customer OAuth involved).

## 2026-05-26 — Overnight cleanup (autonomous)
Jacob slept; ran with explicit overnight autonomy. Per his instruction: only fix confirmed bugs or duplications, don't remove anything, ensure landing + DB are correct in both EN + SV. Acted as senior engineering team on call. Four targeted commits landed; **0 regressions** (1099 tests still pass).

**Commit `72aca63` — legal docs canonicalisation.** All 8 markdown docs under `docs/legal/` (DPA_template_sv, BEHANDLINGSREGISTER, AI_ACT_DOKUMENTATION, DPIA, INTEGRITETSPOLICY, INCIDENT_RESPONSE, SECURITY_CHECKLIST, SLA) referenced the OLD company name `ABN AB` and OLD domain `abnai.se`. Customers downloading the DPA PDF or reading `/legal/<slug>` were getting wrong contact info. Bulk sed substitution: `@abnai.se → @abnplatform.com`, `abnai.se → abnplatform.com`, `ABN AB → ABN Platform AB`. 0 double-substitutions verified. Regenerated `landing/public/legal/dpa.pdf` (12.5 KB) via `scripts/build_dpa_pdf.py`.

**Commit `27daf74` — CI workflow drift + branch-protection doc.** `ci.yml` backend job `name:` was at `Backend — 1018 tests` (real count 1099). Updated, plus the inline comment on line 21 (was still saying "982 tests" from Batch 3 setup). CLAUDE.md `## Branch Protection` updated to current count + added a long-term recommendation to rename the job to count-free `Backend — tests` so the GitHub Settings rule key never has to be manually edited again (drifted 3 times: 982 → 1018 → 1099). Manual GitHub UI step still pending for Jacob.

**Commit `5537406` — drift-proof public test counts.** Three stale public-facing counts:
- `landing/components/SocialProof.tsx:11` — `value: '602'` on the public hero. Real suite is 1099. Replaced with `'1 000+'` (drift-proof — stays accurate as long as suite ≥ 1000).
- `README.md:170` — `'635 passing'` in Status table. Replaced with `'1000+ passing'` + pointer to `JACOB_SESSION.md` for current exact count.
- `README.md:161` — `'13 pages'` for landing. Real is 33. Updated.
- `JACOB_SESSION.md:55` — `'1036 tests passing'` summary line. Updated to 1099. Removed brittle per-batch addition breakdown (dated entries below carry the full accounting).

**Audit findings I did NOT touch (with reasons):**
- **`config.py` `change-me` defaults** (jwt_secret, abn_secret_key, node_signing_key, blueprint_signing_key) — insecure-by-default values that work in dev but allow token forgery in production. Real security risk if `ABN_ENV=production` is set without overriding the env vars. Adding a startup safety check that refuses to boot in prod with defaults is a behaviour change with outage risk — needs explicit Jacob approval.
- **`Org.nr: 559XXX-XXXX` placeholder** in `Footer.tsx:256` + `subprocessors/page.tsx:236`, and `[org. no.]` / `[org.nr]` in 3 legal docs — cannot guess the real Swedish org-number; needs Jacob to provide.
- **`support@abnplatform.com` + `security@abnplatform.com` in `SLA.md`** — per Batch 7 only `hello@`, `reports@`, `legal@` were provisioned on Proton. The two specialised addresses match the canonical pattern but the mailboxes don't exist yet — Jacob needs to either provision them on Proton OR change `SLA.md` to point both at `legal@abnplatform.com`.
- **`report_generator/` vs `dge/` duplication** — both packages produce PDF/Excel/Word/CSV with overlapping APIs. Real duplication per rule #2 but `report_generator/` has live callers; removal is a risky multi-file refactor that needs explicit batch.
- **OPERA legacy consolidation** — still deferred. `test_integration.py` (15 tests) depends on the legacy API surface.
- **Hetzner DB stamp** — manual step documented in CLAUDE.md `## Production DB` since Batch 8.
- **Branch-protection rule key update** — manual GitHub Settings step.
- **`abnai.se` references in CLAUDE.md historical Progress-log entries** — those are historical record of what was created at the time, not active claims. Left untouched.
- **`anna@abnai.se` test fixture in `test_delivery_priority.py`** — fake email in a pseudonymisation test; works regardless of domain.

**Verification before going dark:** backend pytest **1099/1099 pass**, landing `npm run build` **✓ 33/33 pages**, 0 errors, 0 warnings.

## 2026-05-25 — Batch 15 — Semantic Memory + /explain/inquire wired
- **Task 1 — deps:** added `sentence-transformers==3.3.1` + `chromadb==0.5.23` to `backend/requirements.txt`. chromadb installed cleanly in ~1 min; sentence-transformers installed in ~12 min (pulls torch, the heavy one). Both verified importable on this Windows venv.
- **Task 2 — semantic/ package with A→B→C fallback** ✅ done. **Engineering rule #2 design:** the interface is identical across all three layers so callers never branch on availability. `probe_backend()` decides once at `SemanticSearch.__init__` and caches for the instance lifetime; the result rides on every `SearchResult.backend` so the dashboard can display "keyword fallback active" honestly.
  - Layer A: `Embedder` (sentence-transformers, `all-MiniLM-L6-v2`, lazy-loaded — no import-time download) + `VectorStore` (chromadb `PersistentClient` at `ABN_VECTORS_PATH/<tenant_id>/`). Both probe availability via try/import and degrade silently.
  - Layer B: `TfidfBackend` (sklearn, in-memory per-process). Per-(tenant, collection) matrix rebuild on each search — coarse but stable. **sklearn is NOT in requirements.txt** — Layer B only activates if it's installed independently. Documented.
  - Layer C: `KeywordBackend` (pure-Python token Counter + cosine). Zero deps, always available. Smoke-tested before the heavy installs landed.
- **Task 3 — OPERA runner wired:** `_save_run_record` reads back Finding rows for the run and calls `SemanticSearch.index_finding(finding)` for each. Fail-silent — indexing is reporting, not critical path. Runs only when `result.status == "success"`.
- **Task 4 — /explain + /inquire fixed** ✅ done. **Engineering rule #1 catch:** spec said paths `/explain/{finding_id}` and `/inquire`, but the existing routes are `/findings/{finding_id}/explain` + `/findings/{finding_id}/inquire`. Extended the existing routes per rule #2 (no duplicate paths). **Engineering rule #2 catch:** `inquiry_engine/engine.py` already has all the bounded-inquiry logic (7-question budget, phase classification, prompt builder, session management) — wired it up instead of rewriting. The route's `_InquireBody` schema is new; the engine code is untouched. `/explain` returns the dashboard's shape with `why_flagged` / `threshold` / `confidence` / `impact_eur` / `severity` / `source_system` / `related_findings[]` (semantic search hits) / `suggested_questions[]` (4 pre-baked Swedish prompts). `/inquire` enforces the 7-turn budget at the route layer (clean HTTP 429 with the Swedish CTA "Sessionsgräns nådd. Ladda ner rapport för djupare analys.") and again inside the engine via `BudgetExceededError`.
- **Test count: 1075 → 1099** (24 new). 14 in `tests/test_semantic_memory.py` (probe, all three backends' availability checks, keyword upsert + search + tenant isolation, tfidf availability, vector store graceful no-op, SemanticSearch consistent interface, index_finding + index_document_chunk, ranked-result ordering). 10 in `tests/test_explain_inquire.py` (explain shape + DB load + 404, inquire under-budget happy path, at-7-turns 429, history respected, finding_id mismatch 422, unknown finding 404, abstract-only-prompt invariant).
- **2 pre-existing tests updated** to match the new contract: `test_api_routes::test_inquire_valid_phases_pass_schema` and `test_transparency_api::test_inquire_valid_input_reaches_404`. Both were documenting the OLD 404-stub behavior with `finding_id: int` + `phase` body fields. The new schema has `finding_id: str` (since `Finding.id` has been a uuid String column since Batch 1) and phase is now classified inside the engine from `session_history` length, not a body field. Updated tests document the new contract.
- Verification: backend pytest **1099 passed, 0 failures, 125 s**. Landing `npm run build` ✓ 33 static pages, 0 errors.
- CLAUDE.md gains a `## Semantic Memory` section above `## Document Intelligence Engine (DIE)` documenting all 3 layers + the chromadb persistent-path env var + the model choice rationale + the explain/inquire wiring. README updated with a `## Semantic Memory` summary.

## 2026-05-25 — Batch 14 — Document Intelligence Engine
- **Task 1 — deps:** added `pdfplumber==0.11.4`, `pandas==2.2.3`, `pytesseract==0.3.13` to `backend/requirements.txt`. **Engineering rule #6 deferral:** the spec also listed `sentence-transformers==3.3.1` + `chromadb==0.5.23` but neither is used in this batch (they're for Batch 15 — Semantic Memory). Adding them now would pull PyTorch (~1GB+) into every install just to sit unused, plus risk version conflicts in the already-pinned stack. Mirrored the spec's own "DO NOT add whisper yet" logic — deferred to Batch 15.
- **Task 2 — DIE package:** new `backend/die/` package. `models.py` defines `DocumentChunk` (source_path, source_type, content, page_or_sheet, metadata). `registry.py` routes by extension via `_EXT_MAP` (.pdf → PDFReader, .xlsx/.xls → ExcelReader, .docx/.doc → WordReader, .csv → CSVReader, .txt/.md → TextReader, .png/.jpg/.jpeg → ImageReader). Unknown extensions return `[]` with a logged warning — readers never raise on bad input. Six readers in `die/readers/`: PDFReader (pdfplumber, one chunk per page, flags pages with no text layer), ExcelReader (openpyxl read_only, one chunk per sheet, includes columns + row_count), WordReader (python-docx, separate chunks for paragraphs and each table, legacy `.doc` falls back gracefully), CSVReader (pandas, first 1000 rows hard cap, tab-separated chunk content), TextReader (UTF-8 + Latin-1 fallback), ImageReader (pytesseract with `TesseractNotFoundError` catch + `ocr_available=False` metadata flag).
- **Task 3 — file_system action wired:** `action_map.py` already declared `file_system` under `fetch_data` (was a stub). New `agent_engine/capabilities/file_reader.py` provides the runtime `read_document(context, adapters) → context` function. Resolves `file_path` from context or adapters; on success populates `context["document_chunks"]`, `context["document_text"]`, `context["document_chunk_count"]`. Registered four registry keys: `fetch_data.file_system`, `file_system`, `read_document`, `read_file` — all routing to the same function.
- **Task 4 — generate_word + generate_csv exposed:** **Engineering rule #1 catch:** the spec said "Only generate_pdf + generate_excel are declared" — true at the *capability* level, but `action_map.py` already had `word` and `csv` declared. Just needed to expose them as registered capabilities. Added `generate_word` + `generate_csv` to `capabilities/reporting.py` matching the existing SYNC pattern (spec said `async def`, but the existing `generate_pdf`/`generate_excel` are sync — followed rule #2 for consistency across the module). Both queue an `artifact` dict with `kind` = `docx` / `csv` for the A-phase delivery dispatcher. Registered under both canonical (`report.generate_word`, `report.generate_csv`) and legacy short names (`generate_word`, `generate_docx`, `generate_csv`).
- **Task 5 — tests:** 17 tests in `tests/test_die_readers.py`. Synthesise tiny test files on the fly via reportlab/openpyxl/python-docx/Pillow (no fixture files needed). Coverage: each reader's happy path + graceful OCR fallback + registry routing + unknown extension empty list + missing file empty list + supported_extensions surface + capability context population + capability noop without file_path + canonical+legacy name resolution + generate_word/csv artifact shapes. Total backend: **1075 passed** (1058 + 17).
- **Verification:** landing build still ✓ 33 static pages, 0 errors. Backend pytest 95 s, no regressions.
- CLAUDE.md gains a `## Document Intelligence Engine (DIE)` section above `## LLM Policy Modes` covering the package layout, routing table, OCR fail-soft contract, capability wiring, and the explicit Batch 15 deferral of sentence-transformers + chromadb. README updated with a DIE summary section pointing at CLAUDE.md.

## 2026-05-25 — Batch 13B — LLM policy landing + complete log_llm_call wire-up
- **Task 1 — `/company/security` LLM-policy section** ✅ done. Added a `## LLM-datapolicy` DetailSection between "PII Guardian → Result Attestation Layer" and "Compliance". Four cards rendering the 4 policy modes (redacted / no_data / local_only / full). **Engineering rule #1 flag:** user spec said "Light theme (bg-bg text-ink) consistent with page" but `/company/security` uses `DetailPageShell` which is dark-themed (`#0A0A0F` bg + `#F0EEF8` text). Matched the existing dark aesthetic instead — a light-themed section would have looked broken on the dark page. `local_only` card highlighted with `border-2 border-[#6A4BD4]` + `bg-[#1A1430]` + `shadow-glow` + a "Mest säkert →" pill (per spec). Section text emphasises fail-closed contract + the per-customer toggle.
- **Task 2 — wire log_llm_call into the 3 remaining sites** ✅ done. Per spec rule (no DB-arg cascade), each site opens an internal `SessionLocal()` for the transparency write. `industry_detector._detect_industry_llm` logs `task_type="industry_detection"` with the graph's event-type names + node/edge counts. `llm_phase_b._llm_phase_b` logs `task_type="blueprint_generation"` with capability names + archetype/industry/domain context. `dual_brain_critic._llm_critique` logs `task_type="dual_brain_critic"` with blueprint meta + issue count. All three wrap the transparency write in `try/except: pass` so a logging failure never crashes the LLM call itself.
- **Engineering rule #2 flag:** made `tenant_id` optional in `log_llm_call` (default `""`) since the 3 newly-wired sites don't all have tenant context in scope. `ABNLLMCall` model doesn't have a `tenant_id` column anyway — the table is global per-Node and the Node serves exactly one tenant by deployment, so the kwarg is labeling only. Backwards-compatible: existing callers (Batch 13A's OPERA wire) still pass tenant_id explicitly.
- **Path corrections from user spec:** real paths are `blueprint_generator/industry_detector.py` (not `backend/industry_detector.py`), `blueprint_generator/llm_phase_b.py` ✓, and `agent_runtime/execution/dual_brain_critic.py` (not `agent_runtime/dual_brain_critic.py`). Also: `dna_phase/industry_detector.py` exists as a separate file but it's a different concern (DNA-phase industry detection, not the gateway-driven one).
- 4 new tests in `tests/test_transparency_llm.py`: `test_log_llm_call_tenant_id_optional` (the kwarg default) + one per wired site (`test_industry_detector_logs_llm_call`, `test_llm_phase_b_logs_llm_call`, `test_dual_brain_critic_logs_llm_call`). Initial mocking pattern (`patch.object(mod, "httpx", create=True)`) didn't intercept because the sites do `import httpx` INSIDE the function — switched to `patch("httpx.post", ...)` at module level, which is the correct interception point.
- Backend suite: **1058 passed** (1054 + 4), zero regressions. Landing build: ✓ 33 static pages, 0 errors.
- CLAUDE.md `## LLM Policy Modes` section updated: replaced the "wired into OPERA only" note with a 4-row table covering all 4 wired sites + task_type + DB scope, and a note about the SessionLocal-vs-arg-cascade tradeoff.

## 2026-05-25 — Batch 13A — local_only Ollama + abn_llm_calls transparency
- **Task 1 — Ollama provider in the gateway** ✅ done. The user spec said `providers/ollama.py` + `providers/router.py` subdir, but the existing layout is a single `providers.py` module already containing `LocalProvider` + `AnthropicProvider` and a `call_llm` router function. Per rule #2 (no duplication), I extended the existing file with a new `OllamaProvider` class (`/api/chat` endpoint, OpenAI-compatible, 60 s timeout, `OllamaUnavailableError` exception on failure) and updated the routing in `call_llm`: `local_only` mode now returns `OllamaProvider().call(...)` directly with NO try/except — the error propagates so the No-Data Guarantee can't be silently broken by a fallback to the cloud or the deterministic stub. The previous routing also went to `LocalProvider` (the stub) — that was a contract bug; fixed.
- **Task 2 — abn_llm_calls writer + transparency endpoint** ✅ done. **Architectural call:** the user spec said "Add writer in abn-llm-gateway", but the `abn_llm_calls` table lives in the **backend** (the gateway is a stateless service that doesn't know which tenant a call belongs to; the backend owns the tenant context and the customer DB). Wrote the writer in the backend (`agent_runtime/llm_call_log.py`) as the single source of truth per rule #2. The three No-Data invariants (`raw_values_sent` / `customer_data_sent` / `pii_sent`) are always 0, hardcoded by the helper — caller can't override. New `GET /api/transparency/llm-calls` endpoint surfaces the rows with reverse-chronological order + 500 server-side cap. Wired into `OPERARunner._phase_reason` (the most-trafficked LLM call site) — the three other gateway-caller sites (`industry_detector`, `llm_phase_b`, `dual_brain_critic`) don't currently have a DB session in scope; wiring them needs a small refactor and is queued for Batch 13B.
- **Engineering rule #1 catch:** Polite refusal of `providers/` subdirectory layout per the existing single-file `providers.py` (rule #2: no duplication). Polite refusal of "writer in gateway" architecture per tenant-context separation (writer goes in the backend where tenant_id is known).
- **Engineering rule #6 catch:** The pre-Batch-13A `LocalProvider` stub was being used as the `local_only` provider, which violated the policy mode's contract — `local_only` is supposed to mean a real local LLM, not a hardcoded stub return. Test `test_local_only_skips_cloud_fallback_even_with_api_key` now guards against the regression.
- Env vars added to `backend/.env.example`: `OLLAMA_BASE_URL`, `OLLAMA_MODEL`, `OLLAMA_TIMEOUT_SECONDS`, `LLM_POLICY_MODE`. Mirrored on `Settings` in `backend/core/config.py`.
- **Test count: 1048 → 1054 backend** (6 new transparency tests) + **gateway 16 → 21** (5 new Ollama tests). All 33 landing pages still build clean. Zero regressions.
- CLAUDE.md gains a full `## LLM Policy Modes` section above `## Production DB` covering the 4-mode dial, the `local_only` fail-closed contract, env vars, and the `abn_llm_calls` transparency endpoint shape. README updated with a `## LLM Policy Modes` summary that points at CLAUDE.md.

## 2026-05-25 — Batch 12 — schema drift cleanup + OPERA hardening
- **Task 1 — schema drift migration** ✅ done. `alembic revision --autogenerate -m "align_schema_drift"` produced exactly the 6 expected items: missing `index=True` indexes on `findings.{agent_id,tenant_id}`, `invoice_log.tenant_id`, `agent_activity_log.{agent_id,run_id,tenant_id}`; the Subscriptions unique-constraint → unique-index normalisation; and the `tenants.policy` + `tenants.delivery_config` JSON columns. Cleaned up the auto-generated docstring + comments, kept the upgrade/downgrade pair. Migration `8f8acda9625d_align_schema_drift` (depends on `2434b34b2053`). Applied locally; verification: a second autogen against the now-current DB produced an empty diff (deleted that throwaway revision). All 1048 tests still green.
- **Task 2 — OPERA consolidation** ⚠️ DEFERRED again — STOP triggered per user rule #9. After reading all 646 lines of `test_integration.py` end-to-end: **15 tests** depend on the legacy runner, not 7 (TestOperaLoopIntegration 10 + TestBatchRunnerIntegration 3 + TestSecurityIntegration 2). The API gap is structural, not cosmetic: legacy is **sync + accepts `db=None` + uses `AgentRunInput`/`AgentRunOutput` + has `BatchRunner` + raises `SecurityError`**; production is **async + requires a real `Session` + uses `AgentBlueprint`/`OPERARunResult` + has no batch primitive**. Migrating those 15 tests means writing async wrappers + per-run blueprint construction + DB session fixtures + synthesized `evolution_stage` / `culture_rules_applied` fields — a real refactor with non-trivial regression risk against the 1048-test bar Jacob set. Per user rule #9 (*"If tests drop below 1048: STOP, revert, report — do not push"*) and Engineering rule #6 (fail-closed), deferred. **Hardened the legacy as much as possible without rewriting**: added a runtime `DeprecationWarning` on `import opera.integrated_runner` so any new production caller surfaces immediately in CI's warnings summary + test sessions. Same import-line guard banner from Batch 11A is kept on top. CLAUDE.md tech-debt note expanded with the full API-gap audit so the next batch knows exactly what's required.
- **Task 3 — production DB stamp on Hetzner** ⚠️ DOCUMENTED, NOT EXECUTED. No SSH access from this dev environment. Added a `### Current head — verify before stamping` subsection to CLAUDE.md `## Production DB` with the current head revision id (`8f8acda9625d`), the full migration chain (88e0d47ce0ae → c4f1ad9e8a72 → d7e2f9b1c3a4 → e5b8c1f3d29a → 2434b34b2053 → 8f8acda9625d), and the post-stamp verification command (`docker compose exec abn-core alembic current` expecting `8f8acda9625d (head)`). Also added a `### Stamp status` block recording that local dev is stamped + upgraded but Hetzner is still pending. Manual step for Jacob: SSH to `46.225.165.62` and run the documented commands.
- Verification: backend pytest → **1048 passed, 0 failed**; landing `npm run build` → ✓ 33 static pages, 0 errors / 0 warnings.

## 2026-05-25 — Batch 11A Technical debt
- **Task 1 — ROI Ledger writer** ✅ done. New `backend/roi_ledger/{__init__,writer}.py` — `write_roi_entry` is the single source of truth for ROI writes (rule #2). Idempotent per `run_id` (delete-then-insert pattern, mirrors `findings_persistence`); no-op when `impact_eur` is None or 0.0. Wired into `OPERARunner._save_run_record` after the existing AgentRun + Finding + activity-log writes, same try/except defensive pattern, only fires when `result.status == "success"`. New `GET /api/roi/summary` route returns `{total_eur, since_date, days_active, by_agent[], by_month[]}`. Registered in `main.py`. 6 tests in `test_roi_ledger.py`.
- **Task 4 — package-dir duplicates** ✅ done. All 8 nested `pkg/pkg/` dirs deleted (agent_evolution, consistency, delivery, gdpr, inquiry_engine, onboarding, opera, output_schema). Pre-deletion grep confirmed zero imports used the nested `pkg.pkg` form; byte-count showed each pair was identical. Mechanical purge, full suite stayed green.
- **Task 2 — AgentBlackboard persistence** ✅ done. Discovery: the Python `Blackboard` class in `agent_engine/blackboard.py` was already wired to import `AgentBlackboard` from `database.models`; the table simply didn't exist so every `get_unread()` caught `OperationalError` and returned `[]` (CLAUDE.md's "degrades gracefully to empty results"). Added the model with the columns the class already wrote (tenant_id, from_agent, to_agent NULL=broadcast, message_type, payload JSON, created_at, read_by JSON default [], expires_at) + `ix_blackboard_tenant_created` composite index. `created_at` kept as String (codebase convention for ISO timestamps; matches Agent / AgentRun / AgentActivityLog), not DateTime as the user spec wrote. Alembic revision `2434b34b2053_add_agent_blackboard_table` (depends on `e5b8c1f3d29a`). 6 tests in `test_blackboard.py`.
  - Alembic note: autogen detected 4 unrelated drift items (indexes on findings/invoice_log/agent_activity_log, Subscriptions unique→indexed, Tenant.policy + delivery_config columns). Deliberately trimmed those out of the migration so the `-m "add_agent_blackboard_table"` name matches its actual contents. The drift is real schema pre-existing this batch — flagged for a dedicated "align migrations to current models" follow-up batch.
  - Migration apply: local Postgres was at `88e0d47ce0ae` (initial schema) with all later tables physically present (from a prior `create_all`) but `alembic_version` out of date. Ran `alembic upgrade head` to apply the 3 backlog migrations (Findings, Settings+Activity, Subscriptions) properly + the new Blackboard one. Same situation CLAUDE.md documents for the Hetzner production DB — except production needs `alembic stamp head` first because the tables already exist there too.
- **Task 3 — OPERA consolidation** ⚠️ DEFERRED. The legacy `IntegratedOperaRunner` has a fundamentally different API surface (`AgentRunInput`/`AgentRunOutput` dataclasses, `BatchRunner`, `SecurityError`) than the production `OPERARunner` (`AgentBlueprint`/`OPERARunResult`). 7 cases in `test_integration.py` exercise that specific surface. The user's spec said "update import to use agent_engine.opera.runner instead" but that's not a 5-minute substitution — it's a real refactor of the integration test suite. Per Engineering rule #3 (one fix at a time, verify after each) + rule #6 (fail-closed) the consolidation was deferred. As a partial measure: added a "DO NOT IMPORT FROM PRODUCTION CODE" guard banner at the top of `opera/integrated_runner.py` so the legacy doesn't accidentally grow new callers while the consolidation is pending. CLAUDE.md tech-debt entry updated.
- Test count: **1036 → 1048** (1036 baseline + 6 ROI + 6 Blackboard). Zero regressions. Full suite green in 90.82 s.

## 2026-05-25 — Trust & Compliance batch
- /subprocessors created (GDPR Art. 28 compliance) at `landing/app/subprocessors/page.tsx` with 7-row sub-processor register: Anthropic, Clerk, Nango (self-hosted on Hetzner, EU), Hetzner (EU/Germany), Vercel, Proton (Switzerland w/ EU adequacy), Stripe. Footnote disclosing Nango is self-hosted (avoids misrepresenting Nango Inc.'s US infrastructure as EU).
- /subprocessors.json public twin at `landing/public/subprocessors.json` — same register machine-readable, plus a `.policy` block carrying the 30-day notification commitment + the GDPR Art. 28(2)/(4) legal basis + the canonical contact email + register/DPA URLs.
- DPA PDF generated: `landing/public/legal/dpa.pdf` (12.5 KB) rendered from `docs/legal/DPA_template_sv.md` via new `scripts/build_dpa_pdf.py` (reportlab, no system deps needed). Re-runnable when the markdown changes.
- `/legal/dpa` page (rendered through `app/legal/[slug]/page.tsx`) now shows a `Ladda ner DPA (PDF)` button + `Se vår subprocessorlista →` link when `slug === 'dpa'`. Markdown source stays a pure legal document; UI decorations live in JSX so they're conditional on the route.
- Footer fix A: `LegalFooter`'s `legal@abnplatform.com` mentions promoted from plain text to `<a href="mailto:...">` with the same styling as the brand-column `hello@` link. Single `mailto` constant reused across the EN + SV lines (rule #2).
- Footer fix B: Subprocessorer link added to `Företag` column between Säkerhet and Transparens.
- Footer fix C (Task 4): `/legal` → 308 → `/company/legal-center` redirect added to `next.config.mjs`. `/legal/<slug>` is NOT redirected — every doc page stays addressable. Deleted the now-orphan `landing/app/legal/page.tsx` to keep duplication zero (the user's "undvik dubblicering" directive).
- Engineering rule #1 flag caught + raised before publishing: user listed Nango as "EU" but Nango Inc. is YC W22 USA. Confirmed via AskUserQuestion that ABN self-hosts Nango on the Hetzner box → published as "EU (self-hosted)" with explicit footnote. Avoided silent legal misrepresentation under Art. 28(2)/(4).
- Engineering rule #2 catch: `LEGAL_DOCS` registry is the canonical legal-doc list and was already single-sourced via `lib/legal.ts`, but two parallel renders (`/legal` light index + `/company/legal-center` dark index) duplicated the presentation. Collapsed by 308'ing `/legal` to the canonical dark index. `/legal/<slug>` pages keep the dynamic-route renderer; nothing in the catalogue was duplicated.
- README.md updated: new `## Trust & compliance (public surfaces)` section linking to the five public artifacts + the regenerate-the-PDF command.
- Landing build: ✓ Compiled successfully, ✓ Generating static pages (33/33), 0 errors / 0 warnings. Net pages: 33 (added /subprocessors, deleted /legal — exactly balanced).

## 2026-05-25 — Legal page heading readability fix
- Root cause: `landing/app/layout.tsx` `<body>` carries an inline `style={{ backgroundColor: '#0A0A0F', color: '#F0EEF8' }}` (dark theme). Inline styles beat Tailwind utility classes in specificity, so the body's near-black background bled through to any descendant that didn't paint its own bg. The `/legal` index + `/legal/[slug]` markdown pages were designed light (cards `bg-card`=#FFFFFF, borders `border-line`=#E6E3DC, comments in tailwind.config.ts say "warm off-white") and used `text-ink` (#1A1718) headings — which rendered ~1.1:1 contrast against the body's #0A0A0F. Effectively invisible.
- Chose option B: add `bg-bg text-ink` to the outer `<div>` of both legal page files. Single-token additions, no other style touched. Restores the intended light off-white canvas (#FAF9F6) on the legal subtree only — `/company/legal-center` (which uses `DetailPageShell` with its own dark bg) is unaffected.
- Verified: `npm run build` → ✓ Compiled successfully, ✓ Generating static pages (33/33), 0 errors / 0 warnings. `git diff --stat` shows exactly 2 files changed, 2 insertions, 2 deletions.

## 2026-05-25 — Batch 10 — DNS + public API endpoint
- Engineering rule #1/#2: `cors_origins` already exists on `Settings` with a `cors_origins_list` property and `main.py` already reads from it. The spec asked for a parallel `allowed_origins` — kept `cors_origins` as the single source of truth, extended its default to include `https://www.abnplatform.com` + `https://abnplatform.com`. No `ALLOWED_ORIGINS` env var added; `CORS_ORIGINS` is the canonical name.
- `landing/next.config.mjs` (not `.js` or `.ts`) — edited in place, added five security headers via `async headers()`.
- `landing/vercel.json` (NEW): apex `abnplatform.com` → `https://www.abnplatform.com` 308 via `redirects[]` with `has.host` matcher.
- `Settings.public_api_url` (NEW) + surfaced on `GET /health` (`"public_url"` field). `.env.example` carries `PUBLIC_API_URL=https://api.abnplatform.com`.
- CORS middleware: added empty-list `["*"]` fallback so a dev run with no `.env` doesn't 502 the dashboard.
- Task 6 (hardcoded `localhost:8000` / `localhost:3000` / `abn-nine.vercel.app` replacement) — 0 matches in landing. Nothing to fix.
- CLAUDE.md gains `## DNS & Domains` with the manual registrar steps (Vercel A `76.76.21.21`, CNAME `cname.vercel-dns.com`, Hetzner A `46.225.165.62`) + the manual Vercel project step (add both domains under `abn-nine`).
- Backend: 1036 passed (unchanged). Landing: ✓ 33 static pages, 0 errors / 0 warnings.

## 2026-05-25 — Batch 9 — full automation chain
- Five spec-vs-reality conflicts caught pre-code: threshold (existing AUTO=0.85/VERIFIED=0.70 wins over spec's "≥ 0.6"); DNAPhaseOrchestrator signature; ProcessGraphRunner arg order; async-vs-sync chain; CONNECTOR_RESOURCES duplication risk.
- Connector activate now enqueues `_trigger_automation_chain` (DNA → Graph → Agent) as a background task. Each step try/except — fresh-install tenant fails defensively, activation response stays 200.
- New `agent_engine/scheduler.py::AgentScheduler` mirrors ObserverScheduler. Daemon thread, 60 s tick, croniter, per-tenant lock. Wired into main.py lifespan after the Observer scheduler.
- Plan limits via `enforce_agent_limit_or_raise(db, tenant_id)` in agents.py. Starter=2, Professional=25, Enterprise=10000. No Subscription row defaults to starter. Gate on the non-idempotent `generate-from-process` path only.
- Observer `CONNECTOR_RESOURCES` derived from YAML catalogue (`_build_connector_resources` + `_seconds_to_cron`); all 18 catalogue types covered automatically; outbound-direction YAMLs yield empty resource lists; legacy `gps` preserved as fallback.
- 18 new tests in `test_automation_chain.py`. Backend: 1036 passed, 84.45 % coverage. Two bugs caught + fixed in `_is_due` during test runs (window check + cron-validation ordering).

## 2026-05-25 — Batch 8 — migration deploy gap fixed
- Audit (prior turn) surfaced: NO script, Makefile target, or compose/systemd/k8s step ran `alembic upgrade head` on the live server. FastAPI startup called `init_db()` → `Base.metadata.create_all()`, which creates missing tables but never replays migration history or updates `alembic_version`. Four alembic revisions on disk (initial schema + Findings + Settings/Activity + Subscriptions) had only ever been run locally.
- Fix: `backend/main.py` lifespan Step 1 now imports `alembic.config.Config` + `alembic.command` and runs `command.upgrade(alembic_cfg, "head")` instead of `init_db()`. The `init_db()` function itself stays — `scripts/setup.sh` still uses it for fresh local dev installs, and `database/init_db.py` is otherwise unchanged.
- Engineering rule #1 grep: confirmed `init_db` only imported/called in `main.py` + referenced by a (now-stale) comment in `sidecar_entry.py`. Updated the sidecar comment to match.
- Tests: 1018 passed, 84.35% coverage. Verified by grep that no test imports `main.app` — every test builds a fresh `FastAPI()` and overrides `get_db` with an in-memory SQLite. The lifespan change is invisible to the test surface.
- Documented in CLAUDE.md `## Production DB — one-time stamp (MANUAL — already done or do now)`: Jacob must run `docker compose exec abn-core alembic stamp head` once on the Hetzner host (`ssh root@46.225.165.62`, `cd /opt/abn/backend`) before the first restart that contains this change. Reason: the Hetzner DB was bootstrapped by `create_all`, so `alembic_version` is empty/absent — without a stamp, the first `upgrade head` would try to recreate every existing table and fail. After stamping, every future restart runs `alembic upgrade head` and is a no-op until a new revision lands.
- Also fixed: `docs/deployment.md` line 260's aspirational "Database migrations run automatically on startup" — now accurate.
- Flagged but not changed: `Config("alembic.ini")` is CWD-relative. Resolves correctly under Docker (`WORKDIR=/app`) and systemd (`WorkingDirectory=/opt/abn/backend`). The PyInstaller desktop sidecar bundle does NOT include `alembic.ini` + `alembic/versions/`, so a fresh desktop install would fail on first run — `abn-core.spec` needs a `datas=[('alembic.ini', '.'), ('alembic/versions', 'alembic/versions')]` line. Separate batch.

## 2026-05-25 — CI/sync fixes (3 issues from prior diagnostic)
- `ci.yml` backend job `name:` flipped `Backend — 982 tests` → `Backend — 1018 tests`. Branch-protection required-check key matches this literal string, so a manual step is now pending in GitHub Settings → Branches → main → Edit → Required status checks (remove old name, add new). Logged in CLAUDE.md `## Branch Protection` under **MANUAL STEP REQUIRED**.
- `ci.yml` pytest command: dropped `-v`. With 1018 tests this strips ~1018 lines per CI log. Local verify after the change: `1018 passed, 84.35% coverage` against the `--cov-fail-under=80` gate.
- `sync-session-memory.yml`: added `CLAUDE.md` to `paths:` trigger and a `concurrency: { group: sync-session, cancel-in-progress: false }` block (serializes rapid pushes — losers wait, the next push's content is what we want pushed).
- Flagged but NOT changed (no user spec for it): the sync workflow now triggers on `CLAUDE.md` edits but the workflow body only `cp`s `JACOB_SESSION.md` + `CHAT_LOG.md`. So a CLAUDE.md-only edit fires a no-op run (the idempotency guard `git diff --staged --quiet` makes it harmless). If the public mirror should also carry `CLAUDE.md`, the workflow needs an extra `cp` line — separate ask.

## 2026-05-25 — `## JUST NU` cursor rule
- New convention: the first thing after the H1 in JACOB_SESSION.md is a 3-line `## JUST NU` block — Status / Senast / Nästa. Replaced (not appended to) after every batch. The rest of the file stays as the deep-dive history; `JUST NU` is the live cursor a fresh Claude chat reads to know where we are in one glance.
- Rule landed in CLAUDE.md under `## Session Management` so future sessions follow it automatically. Initial block reflects Batch 7 (klar) → Batch 8 (DNS www.abnplatform.com).

## 2026-05-25 — Batch 7 — MarketplacePage wired to real backend
- Batch 7: confirmed Connector table IS the tenant-system join. No new models needed. Wired MarketplacePage to existing POST /connectors/{type}/connect + PATCH endpoints. 8 new tests.
- Renumbering note: the production-deploy work originally logged here as "Batch 7" was relabeled to an "Infra event" (still dated 2026-05-25) so the Batch 7 slot could hold the MarketplacePage wire-up per the user's explicit numbering. Feature batches are now: …Batch 6 (Stripe) → Batch 7 (MarketplacePage). Production deploy stays in the dated history but is not numbered as a feature batch. Next slot: Batch 8 — www.abnplatform.com DNS + api.abnplatform.com.

## 2026-05-25 — Production deploy (infra event, formerly logged as Batch 7)
- Proton Mail domain `abnplatform.com` fully configured: VERIFIED + MX + SPF + DKIM + DMARC all green. Three mailboxes live: `hello@`, `reports@`, `legal@`.
- Vercel DNS fixed: both apex (`abnplatform.com`) and `www.abnplatform.com` report Valid Configuration. Landing continues to auto-deploy from `origin/main` via the Jacob-authored post-merge hook.
- Hetzner CAX21 server `abn-core` provisioned (IP `46.225.165.62`, Frankfurt, Ubuntu 24.04). GDPR DPA signed with Hetzner.
- Docker + Docker Compose installed; repo cloned to `/opt/abn`. All four containers up: `abn-postgres`, `abn-core`, `abn-llm-gateway`, `abn-security`.
- Nginx + Let's Encrypt TLS in front of `abn-core`. `https://api.abnplatform.com/health` returns `{"status":"ok","env":"production"}`.
- Implication for Batch 6 Stripe: the `https://api.abnplatform.com/api/billing/webhook` URL Stripe needs in its dashboard is now real and reachable — no further infra blocker for the Stripe webhook subscription step in `## Manual prereqs for Stripe to go live`.

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
