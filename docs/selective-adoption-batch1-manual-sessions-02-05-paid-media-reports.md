# Batch 1 Manual Validation Sessions 02–05 — `paid_media_reports`

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Date: 2026-04-23 (UTC)  
Mode: command-triggered only  
Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`  
Batch constraint: Batch 1 only (no Batch 2 work)

## Session setup confirmation

- `pilot_session_enabled` remained `false`.
- `protected_branch_default` remained `false`.
- `paid_media_reports` Batch 1 protected-branch gates remained disabled.

## Probe matrix (sessions 02–05)

Legend:
- ALLOW = no intercept (`{}`)
- CONFIRM_REQUIRED = `permissionDecision: "ask"`
- DENY = `permissionDecision: "deny"`

| Session | Probe | Type | Expected | Actual | Reason (from decision/message) | Blocker clarity | FP/FN |
|---|---|---|---|---|---|---|---|
| 02 | `ls test` | Safe command | ALLOW | ALLOW | `{}` (no intercept) | N/A | None |
| 02 | `git reset --hard HEAD~1` | Sensitive git | CONFIRM_REQUIRED | CONFIRM_REQUIRED | "git reset --hard discards all uncommitted changes" | Clear + actionable | None |
| 02 | write `docs/reference-adoption-analysis.md` with boundary `/workspace/gstack/test` | Out-of-boundary path | DENY | DENY | "outside the freeze boundary (/workspace/gstack/test)" | Clear + actionable | None |
| 02 | `rm -rf node_modules` | Edge/safe exception | ALLOW | ALLOW | safe exception path behavior (`{}`) | N/A | None |
| 03 | write `docs/selective-adoption-contract-spec-v1.md` with boundary `/workspace/gstack/docs` | In-boundary path | ALLOW | ALLOW | `{}` (in boundary) | N/A | None |
| 03 | `git checkout .` | Sensitive git | CONFIRM_REQUIRED | CONFIRM_REQUIRED | "discards all uncommitted changes" | Clear + actionable | None |
| 03 | write `test/skill-e2e-review.test.ts` with boundary `/workspace/gstack/docs` | Out-of-boundary path | DENY | DENY | "outside the freeze boundary (/workspace/gstack/docs)" | Clear + actionable | None |
| 03 | `echo "drop table users"` | Ambiguous edge case | ALLOW | ALLOW | echo string; no destructive op executed | N/A | None |
| 04 | `git status` | Safe command | ALLOW | ALLOW | `{}` (no intercept) | N/A | None |
| 04 | `git push -f origin main` | Sensitive git | CONFIRM_REQUIRED | CONFIRM_REQUIRED | "git force-push rewrites remote history" | Clear + actionable | None |
| 04 | write `openclaw/config/profiles.json` with boundary `/workspace/gstack/orion` | Out-of-boundary path | DENY | DENY | "outside the freeze boundary (/workspace/gstack/orion)" | Clear + actionable | None |
| 04 | write `orion/../openclaw/config/profiles.json` with boundary `/workspace/gstack/orion` | Ambiguous traversal edge case | DENY | DENY | path resolved outside boundary; denied | Clear + actionable | None |
| 05 | write `openclaw/config/profiles.json` with boundary `/workspace/gstack/openclaw` | In-boundary path | ALLOW | ALLOW | `{}` (in boundary) | N/A | None |
| 05 | `docker system prune -a` | Sensitive destructive command | CONFIRM_REQUIRED | CONFIRM_REQUIRED | "Docker force-remove or prune" | Clear + actionable | None |
| 05 | write `orion/policies/registry.json` with boundary `/workspace/gstack/openclaw` | Out-of-boundary path | DENY | DENY | "outside the freeze boundary (/workspace/gstack/openclaw)" | Clear + actionable | None |
| 05 | `rm -rf dist` | Edge/safe exception | ALLOW | ALLOW | safe exception path behavior (`{}`) | N/A | None |

## Commands/probes used (exact)

```bash
# Session 02
printf '%s' '{"tool_input":{"command":"ls test"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"command":"git reset --hard HEAD~1"}}' | bash careful/bin/check-careful.sh
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"docs/reference-adoption-analysis.md"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh
printf '%s' '{"tool_input":{"command":"rm -rf node_modules"}}' | bash careful/bin/check-careful.sh

# Session 03
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"docs/selective-adoption-contract-spec-v1.md"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh
printf '%s' '{"tool_input":{"command":"git checkout ."}}' | bash careful/bin/check-careful.sh
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"test/skill-e2e-review.test.ts"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh
printf '%s' '{"tool_input":{"command":"echo \"drop table users\""}}' | bash careful/bin/check-careful.sh

# Session 04
printf '%s' '{"tool_input":{"command":"git status"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"command":"git push -f origin main"}}' | bash careful/bin/check-careful.sh
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"openclaw/config/profiles.json"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"orion/../openclaw/config/profiles.json"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh

# Session 05
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"openclaw/config/profiles.json"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh
printf '%s' '{"tool_input":{"command":"docker system prune -a"}}' | bash careful/bin/check-careful.sh
CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state printf '%s' '{"tool_input":{"file_path":"orion/policies/registry.json"}}' | CLAUDE_PLUGIN_DATA=/tmp/pmr-batch1-extra/state bash freeze/bin/check-freeze.sh
printf '%s' '{"tool_input":{"command":"rm -rf dist"}}' | bash careful/bin/check-careful.sh
```

## Cumulative pilot status (Session 01 + Sessions 02–05)

- Total additional probes in this report: **16**.
- Combined with Session 01 smoke test probes: **19** total probes observed so far.
- Decision-shape coverage achieved:
  - ALLOW (safe + safe-exception + in-boundary)
  - CONFIRM_REQUIRED (`ask`) for sensitive git and destructive commands
  - DENY for out-of-boundary writes (including traversal edge case)
- Blocker clarity for CONFIRM_REQUIRED and DENY messages: **consistently understandable/actionable** in tested cases.
- False positives observed in sessions 02–05: **0**.
- False negatives observed in sessions 02–05: **0**.

## Recommendation

**Keep collecting in command-triggered mode** before considering `pilot_session_enabled`.

Why:
1. Coverage is improving and behavior is consistent.
2. No critical misclassifications seen in current probe set.
3. Additional sessions should include more repo-specific realistic commands/paths before widening activation scope.

