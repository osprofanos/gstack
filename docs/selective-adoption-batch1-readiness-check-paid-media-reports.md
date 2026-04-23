# Batch 1 Pilot Readiness Check — `paid_media_reports`

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Scope reviewed:
1. precedence between repo-local flags, profile activation, and policy registry
2. semantic correctness of `protected-branch-gates.json` `batch1.enabled`
3. boundary preset path correctness
4. threshold completeness (blocker-clarity metric)

## Findings

## 1) Precedence ambiguity across registry/profile/repo flags

**Issue:** No explicit precedence field exists to resolve conflicts between:
- global policy enablement (`orion/policies/registry.json`)
- profile activation defaults (`openclaw/config/profiles.json`)
- repo-local modes (`paid_media_reports/.orion/pilot/batch1-flags.json`)

Current state shows contradictory intent potential:
- registry disables both policies (`enabled=false`)
- profile says command-triggered allowed (`command_triggered=true`)
- repo-local says fully disabled (`disabled=true`, `command_triggered=false`)

Without explicit precedence contract, different orchestrators could interpret activation differently.

**Recommended change (exact fields):**
1. Add `precedence` block to `paid_media_reports/.orion/pilot/batch1-flags.json`:
```json
"precedence": {
  "order": ["kernel", "repo_flags", "profile_activation", "policy_registry"],
  "resolution": "deny_over_allow"
}
```
2. Add `activation_source_of_truth` to `openclaw/config/profiles.json` profile entry:
```json
"activation_source_of_truth": "repo_flags"
```
3. Add optional `activation_guard` to `orion/policies/registry.json` Batch 1 section:
```json
"activation_guard": "repo_flags_required"
```

---

## 2) `protected-branch-gates.json` semantic mismatch for `batch1.enabled`

**Issue:** `openclaw/config/protected-branch-gates.json` has `batch1.enabled=true` while all actual Batch 1 controls are false and repo mode is disabled.

This is semantically misleading: gate section is globally on while controls are effectively off.

**Recommended change (exact field):**
- Set `openclaw/config/protected-branch-gates.json`:
```json
"repos"."paid_media_reports"."batch1"."enabled": false
```
- Transition to `true` only when entering pilot-session or protected-branch-default phases.

---

## 3) Boundary preset path correctness

**Issue:** Preset paths in `paid_media_reports/.orion/pilot/batch1-boundary-defaults.json` are not currently verifiable in this repo checkout (`./reports/`, `./etl/`, `./test/` do not exist under `paid_media_reports/` here).

For Batch 1 command-triggered readiness, boundary presets should point to real, existing directories or be clearly marked as placeholders.

**Recommended change (exact fields):**
Option A (preferred now): mark as placeholders and disable preset selection by default:
```json
"boundary_presets_status": "unverified",
"use_presets_by_default": false
```
Option B: replace paths with verified directories in `paid_media_reports` once available.

---

## 4) Threshold completeness — missing blocker-clarity metric

**Issue:** Current plan thresholds do not explicitly measure whether blocked/confirmed actions are understandable enough for operators to act quickly.

**Recommended addition to Batch 1 thresholds:**
- `blocker_clarity_rate >= 95%`
- Definition: percentage of blocked/confirm-required events where reason + next action are unambiguous in first prompt.
- Validation: weekly sample review (minimum 30 events) with pass/fail rubric.

---

## Readiness decision (command-triggered activation)

**Decision:** **NOT READY** for command-triggered activation yet.

Rationale:
1. Precedence is implied but not codified in config artifacts.
2. `batch1.enabled=true` creates semantic mismatch with disabled controls.
3. Boundary presets are unverified for current pilot repo filesystem.
4. Blocker-clarity metric is missing from readiness gate.

Once the exact field updates above are applied, readiness can be re-evaluated quickly without refactor.

