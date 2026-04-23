# Batch 1 Patch Plan v1 (Pilot: `paid_media_reports`)

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Status: No-code patch planning document  
Applies to: Batch 1 only (`command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`)  
Authority baseline: Orion/OpenClaw arbitration order from Contract Spec v1 remains unchanged.

## Pilot adaptation

This plan narrows the generic implementation plan to the `paid_media_reports` pilot repository.

**Why `paid_media_reports` is suitable for Batch 1**
1. Moderate operational risk: frequent report generation scripts and branch churn make command-safety and write-boundary controls meaningful.
2. Contained blast radius: business-critical but not platform-kernel-critical, enabling safe rollout with fail-fast rollback.
3. Clear ownership: small team workflows make session-level pilot controls practical.

**Pilot assumptions for this repo**
- Default protected branch: `main`.
- Primary writable work area for feature sessions: repo workspace (boundary scoped per task/module, not entire monorepo root by default).
- Existing Orion kernel interceptors already active and authoritative.

---

## File patch map

All paths are additive and limited to required Batch 1 integration points.

## Create
1. `orion/policies/command_safety_gate.contract.json`
   - Contract descriptor only (fields/decisions from Contract Spec v1).
2. `orion/policies/edit_scope_boundary.contract.json`
   - Contract descriptor only.
3. `orion/profiles/change_guard_profile.json`
   - Profile composition: enables both policies atomically.
4. `paid_media_reports/.orion/pilot/batch1-flags.json`
   - Repo-local pilot toggles (`disabled`, `command_triggered`, `pilot_session_enabled`, `protected_branch_default`).
5. `paid_media_reports/.orion/pilot/batch1-boundary-defaults.json`
   - Optional repo-local boundary presets (safe starter scopes).
6. `docs/selective-adoption-batch1-patch-plan-v1-paid-media-reports.md`
   - This plan artifact.

## Modify
1. `orion/policies/registry.json`
   - Register two new policy contracts in disabled state.
2. `openclaw/config/profiles.json`
   - Expose `change_guard_profile` for command-triggered activation in pilot sessions.
3. `openclaw/config/protected-branch-gates.json`
   - Add Batch 1 pilot flags for `paid_media_reports` only (no Batch 2 fields).

## Do not touch (Batch 1 guardrail)
- Any pre-merge review/routing/ledger files.
- Any global workflow stage definitions for Batch 2.
- Any kernel arbitration precedence code.

---

## Patch order (minimum additive)

### P1. Add passive contracts (no runtime effect)
- Create `command_safety_gate.contract.json` and `edit_scope_boundary.contract.json`.
- Register both in `orion/policies/registry.json` with `enabled=false`.

### P2. Add profile composition
- Create `change_guard_profile.json` with atomic enablement of both policies.
- Add profile entry to `openclaw/config/profiles.json`.

### P3. Add pilot repo flags (still no enforcement)
- Create `paid_media_reports/.orion/pilot/batch1-flags.json` defaulting all modes to off.
- Create optional `batch1-boundary-defaults.json` presets.

### P4. Enable command-triggered mode only
- Toggle repo pilot flags: `command_triggered=true`.
- Keep `pilot_session_enabled=false`, `protected_branch_default=false`.

### P5. Enable pilot-session mode
- Toggle repo pilot flags: `pilot_session_enabled=true` for selected sessions/users.
- Keep protected branch default off until thresholds are met.

### P6. Protected-branch default (conditional)
- If thresholds pass, set `protected_branch_default=true` for `main` in this repo only.
- Non-protected branches remain command-triggered.

Rollback-safe property:
- Every step is reversible by flag/profile/registry rollback without structural refactor.

---

## Activation sequence

Sequence is strictly progressive for `paid_media_reports`.

1. **Disabled**
   - Contracts present but not active.
   - Registry entries disabled.
   - Profile available but not auto-applied.

2. **Command-triggered**
   - Operator explicitly invokes `change_guard_profile` per session.
   - No auto-default on any branch.

3. **Pilot-session enabled**
   - Repo pilot flag enables auto-profile for approved pilot sessions only.
   - Session allowlist/team-scoped.

4. **Protected-branch default (if applicable)**
   - Apply default profile on `main` interactions in this repo only.
   - Keep non-protected branches command-triggered.
   - Keep kernel-approved override path unchanged.

---

## Stability thresholds (required before Batch 2)

Evaluate over a minimum 7-day pilot window in `paid_media_reports`.

1. **Policy correctness**
   - 0 kernel precedence violations.
   - 0 cases where policy bypassed a kernel deny.

2. **False positive control**
   - `command_safety_gate` override rate on flagged commands <= 20% after first 3 days.
   - `edit_scope_boundary` deny reversals (user-validated false denies) <= 10% of denies.

3. **Operational friction**
   - No increase >10% in median PR cycle time attributable to Batch 1 controls.
   - No critical delivery incident caused by boundary misconfiguration.

4. **Reliability**
   - Policy decision path success rate >= 99.5% (excluding user-declined confirms).
   - Session profile activation/deactivation success >= 99%.

5. **Auditability**
   - 100% of intercept decisions present in existing Orion event logs for pilot sessions.

If any threshold fails, remain in Batch 1 and do not start Batch 2.

---

## Test plan (repo-specific)

## A. `command_safety_gate` tests in `paid_media_reports`
1. Safe report generation command => `allow`.
2. Dangerous recursive delete in repo workspace => `confirm_required`.
3. Force-push command on feature branch => `confirm_required`.
4. Malformed/suspicious command payload => `deny`.
5. Kernel deny + policy allow scenario => final deny (authority preserved).

## B. `edit_scope_boundary` tests in `paid_media_reports`
1. Write inside selected report module boundary => `allow`.
2. Write outside boundary (e.g., unrelated infra dir) => `deny`.
3. Symlink/path traversal attempt => `deny` (`path_unresolvable` or outside).
4. Boundary unset in command-triggered mode => `allow` only when no stricter repo flag requires boundary.
5. Boundary-required flag with no boundary set => `deny`.

## C. `change_guard_profile` tests in `paid_media_reports`
1. Profile activation enables both policies in one step.
2. Profile deactivation disables both policies cleanly.
3. Boundary not set + profile active => command safety still active; boundary behavior follows Contract Spec v1.

## D. Activation progression tests
1. Disabled mode: no intercept decisions from Batch 1 policies.
2. Command-triggered mode: intercepts only in sessions that invoked profile.
3. Pilot-session mode: intercepts for allowlisted sessions only.
4. Protected-branch default mode: auto-active on `main`; not auto-active on non-protected branches.

---

## Rollback plan

## Immediate rollback (no refactor)
1. Set `protected_branch_default=false` in `paid_media_reports/.orion/pilot/batch1-flags.json`.
2. Set `pilot_session_enabled=false`.
3. Set `command_triggered=false` if needed.
4. Disable policy entries in `orion/policies/registry.json`.
5. Remove `change_guard_profile` from `openclaw/config/profiles.json` exposure (or mark inactive).

## Safe retention
- Keep contract files and this patch-plan doc for traceability.
- Do not delete existing Orion logs.

## Fallback mode
- Return repo to pre-pilot baseline (kernel-only behavior).
- Re-enable command-triggered mode only after issue review and boundary preset correction.

## Rollback completion criteria
- Orion/OpenClaw flows restored to baseline behavior.
- No active Batch 1 policy intercepts in new sessions.
- Pilot incident summary documented with corrective actions.

---

## Minimality / authority checks

- Batch 1 only; no Batch 2 artifacts introduced.
- No parallel second system.
- No refactor outside explicit integration files.
- Orion kernel precedence preserved exactly.
