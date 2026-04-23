# Selective Adoption Implementation Plan v1 (No-Code)

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Status: Execution plan aligned to Contract Spec v1  
Contract baseline: `docs/selective-adoption-contract-spec-v1.md`

## Pilot recommendation

**Pilot repo:** `payments-api` (or equivalent mid-critical service with stable CI + active PR volume).

**Why this is the best first target**
1. Has meaningful risk surface (SQL, shell ops in CI/release scripts, enum/state transitions) so Batch 1+2 controls are exercised realistically.
2. Not the highest-blast-radius platform core, so fail-closed behavior can be validated without organization-wide disruption.
3. Frequent pre-merge activity gives fast signal on false positives, routing quality, and ledger utility.
4. Existing OpenClaw PR workflow already present, minimizing integration work.

**Pilot guardrails**
- Opt-in profile for first week, then protected-branch-only enforcement.
- Single service team ownership + daily review of intercept/route events.

---

## Batch plan

## Batch 1 (Safety primitives)
Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`

### Step B1-0: Preconditions (readiness)
- Confirm Orion kernel interceptor precedence is unchanged.
- Confirm session/profile state store exists and supports session-scoped config.
- Confirm no duplicate existing policy names.

Rollback-safe checkpoint:
- If any precondition fails, stop before creating runtime bindings.

### Step B1-1: Register policy contracts (no enforcement)
- Add contract descriptors for:
  - `command_safety_gate`
  - `edit_scope_boundary`
- Keep in `disabled` state by default.

Rollback-safe checkpoint:
- Removing descriptors is non-breaking (no active references yet).

### Step B1-2: Wire `change_guard_profile` composition
- Add profile entry that atomically enables both policies.
- Ensure boundary root is optional; behavior follows Contract Spec v1 (`no_boundary` semantics).

Rollback-safe checkpoint:
- Profile removal reverts behavior to pre-pilot baseline.

### Step B1-3: Enable in pilot as command-triggered only
- Enable profile for pilot sessions only (manual trigger).
- Log policy intercept outcomes to existing event path.

Rollback-safe checkpoint:
- Toggle profile off; no code-path migration required.

### Step B1-4: Promote to protected-branch profile default (pilot repo only)
- Keep explicit operator override under kernel control.
- Keep non-protected branches command-triggered.

Rollback-safe checkpoint:
- Revert protected-branch default to command-triggered mode.

---

## Batch 2 (Pre-merge review + routing + ledger)
Scope: `premerge_risk_review`, `review_action_router`, `review_event_ledger`

### Step B2-0: Preconditions (Batch 1 stable)
- Batch 1 false-positive/override rate within acceptable threshold.
- No unresolved policy collision issues.

Rollback-safe checkpoint:
- If thresholds not met, keep Batch 2 disabled.

### Step B2-1: Add pre-merge stage contract binding (observe-only)
- Register `premerge_risk_review` stage in OpenClaw pre-merge pipeline.
- Run in observe-only mode; do not block merge yet.
- Categories fixed to Contract Spec v1 critical subset.

Rollback-safe checkpoint:
- Remove stage from pipeline manifest; previous merge path unaffected.

### Step B2-2: Add `review_action_router` binding
- Route each finding to `auto_fix_candidate|ask_user|defer` using contract rules.
- Critical defaults to `ask_user` unless explicit policy exception.

Rollback-safe checkpoint:
- Disable router; findings remain informational.

### Step B2-3: Add `review_event_ledger` append-only writes
- Emit `review_run` and per-finding route/action events.
- Validate write success behavior per Contract Spec (fail-closed in pre-merge once gating enabled).

Rollback-safe checkpoint:
- Keep stage in observe-only if ledger durability is unstable.

### Step B2-4: Turn on fail-closed pre-merge gating (pilot protected branches)
- Enable block-on-failed-review and block-on-ledger-write-failure.
- Keep non-protected branches observe-only for one additional cycle.

Rollback-safe checkpoint:
- Revert to observe-only by config toggle (no artifact deletion required).

---

## File map (expected artifacts)

Note: paths are representative; use existing Orion/OpenClaw directories with equivalent roles.

## New/updated artifacts for Batch 1
- `orion/policies/command_safety_gate.contract.json` (new)
- `orion/policies/edit_scope_boundary.contract.json` (new)
- `orion/profiles/change_guard_profile.json` (new)
- `orion/policies/registry.json` (update)
- `openclaw/config/profiles.json` (update: expose profile)
- `docs/selective-adoption-contract-spec-v1.md` (reference; no semantic changes expected)

## New/updated artifacts for Batch 2
- `openclaw/workflows/premerge.pipeline.json` (update: add `premerge_risk_review` stage)
- `orion/review/premerge_risk_review.contract.json` (new)
- `orion/review/review_action_router.contract.json` (new)
- `orion/audit/review_event_ledger.schema.json` (new)
- `orion/audit/sinks.json` (update: append-only ledger sink)
- `openclaw/config/protected-branch-gates.json` (update: gating mode flags)

## Pilot-local runtime artifacts
- `.orion/audit/review-events.jsonl` (per-project ledger store)
- `.orion/state/change-guard-session.json` (session/profile state)

---

## Integration points

## Orion kernel/governance
- Policy arbitration remains first and authoritative.
- Project overrides only tighten behavior (never weaken kernel decisions).
- Confirm/deny semantics inherited exactly from Contract Spec v1.

## OpenClaw orchestration
- Batch 1: profile exposure and session activation hooks.
- Batch 2: pre-merge stage insertion + protected-branch gating toggles.

## Existing Claude skills/workflows
- No new command namespace required.
- Existing review workflows call into `premerge_risk_review` stage; no second review system introduced.

## Audit/observability
- Reuse existing Orion event sink plumbing; add only compact v1 event payloads.
- Append-only ledger semantics; fail-closed in gated pre-merge contexts.

---

## Test plan

## Batch 1 test cases

### `command_safety_gate`
1. Safe command => `allow`.
2. Known destructive command => `confirm_required`.
3. Parse failure with suspicious payload => `deny`.
4. Kernel deny + policy allow => final deny (precedence).

### `edit_scope_boundary`
1. Inside boundary write => `allow`.
2. Outside boundary write => `deny`.
3. Unresolvable path => `deny`.
4. Boundary required by override but unset => `deny`.
5. No active boundary and no stricter override => `allow`.

### `change_guard_profile`
1. Activation enables both policies atomically.
2. Profile active + boundary unset => command safety active, boundary returns `no_boundary` behavior.
3. Profile deactivation returns to baseline behavior.

## Batch 2 test cases

### `premerge_risk_review`
1. Valid diff/base + no findings => `clean`.
2. Findings present => `issues_found` with proper summary counts.
3. Missing base/diff => `failed` and merge blocked in gated mode.
4. Enum completeness check reads bounded out-of-diff references.

### `review_action_router`
1. Critical finding => `ask_user` default.
2. Mechanical reversible finding (explicit policy mark) => `auto_fix_candidate`.
3. Low confidence finding => `ask_user` or `defer` (never auto-fix).
4. Malformed finding payload => `defer` + warning event.

### `review_event_ledger`
1. `review_run` event append success.
2. Per-finding route event append success.
3. Ledger write failure in pre-merge gated mode => fail-closed.
4. Ledger write failure in observe-only/non-merge mode => warning + continue if policy allows.

---

## Rollback plan

## Immediate rollback (configuration-only)
- Disable `change_guard_profile` default activation.
- Revert `premerge_risk_review` from gated to observe-only.
- Disable protected-branch fail-closed flags.

## Artifact rollback
- Keep contract/schema files in repo (for traceability), but remove from active registries/pipeline manifests.
- Preserve ledger history; do not delete audit logs.

## Fallback operating mode
1. Batch 2 degraded: keep review observe-only, routing optional, ledger best-effort.
2. Batch 1 degraded: keep command safety command-triggered only; disable boundary enforcement defaults.

## Exit criteria after rollback
- Kernel arbitration stable.
- CI and merge flow restored to baseline.
- Incident review completed before re-attempting rollout.

---

## Minimality checks (must remain true)

- No broad refactor.
- No parallel second review system.
- Contract Spec v1 fields and precedence unchanged.
- Pilot-first, protected-branch-first enforcement only.
