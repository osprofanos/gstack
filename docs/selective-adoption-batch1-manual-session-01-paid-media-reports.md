# Batch 1 Manual Validation Session 01 — `paid_media_reports`

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Date: 2026-04-22 (UTC)  
Mode: command-triggered only  
Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`

## Session steps

1. Verified command-triggered-only mode in pilot flags.
2. Verified no auto-enablement via `pilot_session_enabled` and no protected-branch default behavior.
3. Explicitly activated profile for this single validation session (manual simulation marker).
4. Manually selected boundary preset `tests_only` (`./test/`) and set boundary to `/workspace/gstack/test` for the session.
5. Ran three behavior probes:
   - safe command (expect ALLOW)
   - sensitive/destructive command (expect CONFIRM_REQUIRED)
   - out-of-boundary write path (expect DENY)
6. Captured exact decision payloads and evaluated blocker clarity.

## Commands / probes used

```bash
# Session marker
SESSION_DIR=/tmp/pmr-batch1-session
mkdir -p "$SESSION_DIR"
cat > "$SESSION_DIR/session-state.json" <<'JSON'
{"profile":"change_guard_profile","activation":"command_triggered","repo":"paid_media_reports"}
JSON

# Manual boundary selection from preset tests_only -> ./test/
BOUNDARY="/workspace/gstack/test"
mkdir -p "$SESSION_DIR/state"
echo "$BOUNDARY" > "$SESSION_DIR/state/freeze-dir.txt"

# Probe 1: safe command
printf '%s' '{"tool_input":{"command":"echo hello"}}' | bash careful/bin/check-careful.sh

# Probe 2: sensitive/destructive command
printf '%s' '{"tool_input":{"command":"git push --force origin main"}}' | bash careful/bin/check-careful.sh

# Probe 3: out-of-boundary path
CLAUDE_PLUGIN_DATA="$SESSION_DIR/state" printf '%s' '{"tool_input":{"file_path":"docs/reference-adoption-analysis.md"}}' \
  | CLAUDE_PLUGIN_DATA="$SESSION_DIR/state" bash freeze/bin/check-freeze.sh

# Confirm no auto pilot-session/default protected-branch behaviors
jq -c '.modes' paid_media_reports/.orion/pilot/batch1-flags.json
jq -c '.repos.paid_media_reports.batch1' openclaw/config/protected-branch-gates.json
```

## Observed results

```json
{
  "safe_probe": {},
  "sensitive_probe": {
    "permissionDecision": "ask",
    "message": "[careful] Destructive: git force-push rewrites remote history. Other contributors may lose work."
  },
  "deny_probe": {
    "permissionDecision": "deny",
    "message": "[freeze] Blocked: /workspace/gstack/docs/reference-adoption-analysis.md is outside the freeze boundary (/workspace/gstack/test). Only edits within the frozen directory are allowed."
  },
  "pilot_modes": {
    "disabled": false,
    "command_triggered": true,
    "pilot_session_enabled": false,
    "protected_branch_default": false
  },
  "protected_branch_batch1": {
    "enabled": false,
    "command_safety_gate": false,
    "edit_scope_boundary": false,
    "change_guard_profile_default": false
  }
}
```

## Contract Spec v1 match check

- Safe command => ALLOW behavior: **PASS** (`{}` from guardrail check = no block/intercept).
- Sensitive command => CONFIRM_REQUIRED behavior: **PASS** (`permissionDecision: ask`).
- Out-of-boundary write => DENY behavior: **PASS** (`permissionDecision: deny`).
- Command-triggered-only mode, no pilot-session default, no protected-branch default: **PASS**.

## Blocker clarity evaluation

- Sensitive command message was understandable and actionable (clear risk + why confirmation needed).
- Deny message was understandable and actionable (clear path, boundary, and remediation direction).

Blocker clarity result for this session: **PASS**.

## False positives / false negatives

- False positives observed: **none** in these 3 probes.
- False negatives observed: **none** in these 3 probes.

## Recommendation

**KEEP** command-triggered Batch 1 pilot mode and continue with additional sample sessions.

Notes:
- This was one manual validation session; keep collecting blocker-clarity samples before any broader activation changes.
- No Batch 2 actions were performed.
