# Batch 1 Command-Triggered Activation Checklist — `paid_media_reports`

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`  
Mode: **command-triggered only** (no `pilot_session_enabled`, no `protected_branch_default`, no Batch 2)

## 1) Exact config changes to apply

Apply only the edits below.

## A. `paid_media_reports/.orion/pilot/batch1-flags.json`
Set modes to command-triggered state:
- `modes.disabled = false`
- `modes.command_triggered = true`
- `modes.pilot_session_enabled = false`
- `modes.protected_branch_default = false`

Add explicit precedence block (from readiness check recommendation):
```json
"precedence": {
  "order": ["kernel", "repo_flags", "profile_activation", "policy_registry"],
  "resolution": "deny_over_allow"
}
```

## B. `openclaw/config/profiles.json`
For profile `change_guard_profile`:
- keep `activation.default = "disabled"`
- keep `activation.command_triggered = true`
- keep `activation.pilot_session_enabled = false`
- keep `activation.protected_branch_default = false`
- add `activation_source_of_truth = "repo_flags"`

## C. `orion/policies/registry.json`
Keep both Batch 1 policies disabled by default at registry level:
- `batch1.policies[name=command_safety_gate].enabled = false`
- `batch1.policies[name=edit_scope_boundary].enabled = false`

Add guard:
```json
"activation_guard": "repo_flags_required"
```

## D. `openclaw/config/protected-branch-gates.json`
For `repos.paid_media_reports.batch1`:
- set `enabled = false`
- keep `command_safety_gate = false`
- keep `edit_scope_boundary = false`
- keep `change_guard_profile_default = false`

Rationale: command-triggered mode should not imply protected-branch defaults.

## E. `paid_media_reports/.orion/pilot/batch1-boundary-defaults.json`
Mark presets as unverified placeholders until repo paths are confirmed:
```json
"boundary_presets_status": "unverified",
"use_presets_by_default": false
```

Do not auto-apply any preset boundary in first command-triggered session.

---

## 2) Validation commands to run

Run from repo root.

### 2.1 JSON parse validation
```bash
for f in \
  orion/policies/registry.json \
  openclaw/config/profiles.json \
  openclaw/config/protected-branch-gates.json \
  paid_media_reports/.orion/pilot/batch1-flags.json \
  paid_media_reports/.orion/pilot/batch1-boundary-defaults.json; do
  jq empty "$f"
done
```

### 2.2 Command-triggered mode assertions
```bash
jq -e '.modes.disabled == false and .modes.command_triggered == true and .modes.pilot_session_enabled == false and .modes.protected_branch_default == false' paid_media_reports/.orion/pilot/batch1-flags.json
```

### 2.3 Precedence assertions
```bash
jq -e '.precedence.order == ["kernel","repo_flags","profile_activation","policy_registry"] and .precedence.resolution == "deny_over_allow"' paid_media_reports/.orion/pilot/batch1-flags.json
```

### 2.4 Profile source-of-truth assertions
```bash
jq -e '.profiles[] | select(.name=="change_guard_profile") | .activation.default=="disabled" and .activation.command_triggered==true and .activation.pilot_session_enabled==false and .activation.protected_branch_default==false and .activation_source_of_truth=="repo_flags"' openclaw/config/profiles.json
```

### 2.5 Registry guard + disabled policy assertions
```bash
jq -e '.batch1.activation_guard == "repo_flags_required" and (.batch1.policies | all(.enabled == false))' orion/policies/registry.json
```

### 2.6 Protected-branch gate assertions (must be off)
```bash
jq -e '.repos.paid_media_reports.batch1.enabled == false and .repos.paid_media_reports.batch1.command_safety_gate == false and .repos.paid_media_reports.batch1.edit_scope_boundary == false and .repos.paid_media_reports.batch1.change_guard_profile_default == false' openclaw/config/protected-branch-gates.json
```

### 2.7 Boundary preset safety assertions
```bash
jq -e '.boundary_presets_status == "unverified" and .use_presets_by_default == false' paid_media_reports/.orion/pilot/batch1-boundary-defaults.json
```

---

## 3) Expected results

- All `jq empty` checks pass (exit code 0).
- All assertion commands pass (exit code 0).
- Effective mode is command-triggered only:
  - no pilot-session defaults
  - no protected-branch defaults
  - no automatic Batch 1 enablement from protected-branch gate
- Policies remain registry-disabled by default and only usable through explicit command-triggered profile flow.

If any assertion fails, do **not** run pilot session.

---

## 4) Rollback commands/edits

If activation prep causes ambiguity, revert to hard-disabled Batch 1.

### 4.1 Disable repo-local modes
```bash
jq '.modes.disabled=true | .modes.command_triggered=false | .modes.pilot_session_enabled=false | .modes.protected_branch_default=false' paid_media_reports/.orion/pilot/batch1-flags.json > /tmp/batch1-flags.json && mv /tmp/batch1-flags.json paid_media_reports/.orion/pilot/batch1-flags.json
```

### 4.2 Remove precedence block (optional rollback-to-baseline)
```bash
jq 'del(.precedence)' paid_media_reports/.orion/pilot/batch1-flags.json > /tmp/batch1-flags.json && mv /tmp/batch1-flags.json paid_media_reports/.orion/pilot/batch1-flags.json
```

### 4.3 Re-disable profile source-of-truth override
```bash
jq '(.profiles[] | select(.name=="change_guard_profile") | del(.activation_source_of_truth))' openclaw/config/profiles.json > /tmp/profiles.json && mv /tmp/profiles.json openclaw/config/profiles.json
```

### 4.4 Keep registry and protected-branch gates off
```bash
jq '.batch1.activation_guard="repo_flags_required" | .batch1.policies |= map(.enabled=false)' orion/policies/registry.json > /tmp/registry.json && mv /tmp/registry.json orion/policies/registry.json
jq '.repos.paid_media_reports.batch1.enabled=false | .repos.paid_media_reports.batch1.command_safety_gate=false | .repos.paid_media_reports.batch1.edit_scope_boundary=false | .repos.paid_media_reports.batch1.change_guard_profile_default=false' openclaw/config/protected-branch-gates.json > /tmp/pbg.json && mv /tmp/pbg.json openclaw/config/protected-branch-gates.json
```

Re-run validation commands after rollback.

---

## 5) Operator checklist for first pilot session

Use this checklist in order.

1. Confirm config assertions all pass (Section 2).
2. Confirm session goal is narrow and set a manual boundary (do **not** rely on presets).
3. Explicitly activate `change_guard_profile` via command-triggered flow.
4. Run one safe command (expect allow).
5. Run one known-destructive dry-run command shape (expect confirm_required, then cancel).
6. Attempt one write outside chosen boundary (expect deny).
7. Attempt one write inside chosen boundary (expect allow).
8. Confirm no pilot-session auto-enablement occurred.
9. Confirm no protected-branch default activation occurred.
10. Record outcomes and any unclear blocker messages for blocker-clarity metric sampling.

Exit criteria for session #1:
- All three expected control behaviors observed (`allow`, `confirm_required`, `deny`).
- No precedence anomaly vs kernel controls.
- No automatic activation outside command-triggered scope.

