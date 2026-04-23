# Scoped Change Control — STABLE / CONTROLLED MODE (System Reference)

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Status: **stable_controlled**  
Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`  
Activation posture: manual only (command-triggered), no automatic activation.

## What Batch 1 does

Batch 1 provides two safety primitives and one composed profile:

1. `command_safety_gate`
   - Pre-execution command safety intercept for risky/destructive operations.
2. `edit_scope_boundary`
   - Boundary enforcement for write/edit targets.
3. `change_guard_profile`
   - Manual profile that enables both controls together for a single session.

## When to use it

Use Batch 1 when an operator needs extra protection during focused change sessions,
especially when commands can alter history/state or when edits must stay inside a
strict boundary.

## How to activate it manually

1. Ensure repo flags are in command-triggered mode:
   - `command_triggered=true`
   - `pilot_session_enabled=false`
   - `protected_branch_default=false`
2. Explicitly invoke `change_guard_profile` for the current session.
3. Select boundary manually (do not auto-apply presets).

## Integration guarantees

- Batch 1 is globally available in Orion/OpenClaw metadata.
- Batch 1 is **not** auto-activated for any repo/workflow by default.
- Orion kernel precedence remains authoritative.
- No Batch 2 behavior is introduced by this mode.
