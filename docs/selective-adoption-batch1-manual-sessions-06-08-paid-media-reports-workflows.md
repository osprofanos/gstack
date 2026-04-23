# Batch 1 Manual Validation Sessions 06–08 — `paid_media_reports` (Workflow-Oriented)

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Date: 2026-04-23 (UTC)  
Mode: command-triggered only  
Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`  
Constraints honored: Batch 1 only, no protected-branch default, no Batch 2, no refactor.

## Session setup / mode checks

- `pilot_session_enabled` remained `false`.
- `protected_branch_default` remained `false`.
- `protected-branch-gates.batch1.enabled` remained `false`.

## Workflow sessions and probe outcomes

Legend:
- ALLOW = `{}`
- CONFIRM_REQUIRED = `permissionDecision: "ask"`
- DENY = `permissionDecision: "deny"`

| Session | Workflow action | Expected | Actual | Reason | Blocker clarity | Workflow friction | FP/FN |
|---|---|---|---|---|---|---|---|
| 06 | Report file op inside reports boundary (`paid_media_reports/reports/daily_spend_report.sql`) | ALLOW | ALLOW | in-boundary (`{}`) | N/A | None | None |
| 06 | Normal git action (`git status`) | ALLOW | ALLOW | safe command (`{}`) | N/A | None | None |
| 06 | Risky git action (`git push --force-with-lease origin feature/pmr`) | CONFIRM_REQUIRED | CONFIRM_REQUIRED | force-push risk warning | Clear + actionable | Low (expected gate) | None |
| 06 | ETL file path outside reports boundary (`paid_media_reports/etl/load_pipeline.py`) | DENY | DENY | outside freeze boundary | Clear + actionable | Low (expected deny) | None |
| 06 | Edge case: commit message contains `DROP TABLE` text (`git commit -m "DROP TABLE cleanup note"`) | ALLOW | ALLOW | non-destructive command form; no intercept | N/A | None | None |
| 07 | ETL file op inside etl boundary (`paid_media_reports/etl/normalize_costs.py`) | ALLOW | ALLOW | in-boundary (`{}`) | N/A | None | None |
| 07 | Normal git workflow step (`git add paid_media_reports/etl/normalize_costs.py`) | ALLOW | ALLOW | safe command (`{}`) | N/A | None | None |
| 07 | Risky git action (`git checkout .`) | CONFIRM_REQUIRED | CONFIRM_REQUIRED | discards uncommitted changes | Clear + actionable | Low (expected gate) | None |
| 07 | Report file path outside etl boundary (`paid_media_reports/reports/weekly.md`) | DENY | DENY | outside freeze boundary | Clear + actionable | Low (expected deny) | None |
| 07 | Edge traversal path (`paid_media_reports/etl/../reports/weekly.md`) | DENY | DENY | normalized path outside boundary | Clear + actionable | Low (expected deny) | None |
| 08 | Report build step (`python -m py_compile paid_media_reports/reports/build_report.py`) | ALLOW | ALLOW | safe command (`{}`) | N/A | None | None |
| 08 | Risky git action (`git reset --hard HEAD~1`) | CONFIRM_REQUIRED | CONFIRM_REQUIRED | hard reset data-loss warning | Clear + actionable | Low (expected gate) | None |
| 08 | Report file op inside reports boundary (`paid_media_reports/reports/build_report.py`) | ALLOW | ALLOW | in-boundary (`{}`) | N/A | None | None |
| 08 | Out-of-boundary docs target (`docs/selective-adoption-implementation-plan-v1.md`) | DENY | DENY | outside freeze boundary | Clear + actionable | Low (expected deny) | None |
| 08 | Edge safe exception (`rm -rf dist`) | ALLOW | ALLOW | safe exception pattern (`{}`) | N/A | None | None |

## Commands/probes used (exact)

```bash
# Session 06
printf '%s' '{"tool_input":{"file_path":"paid_media_reports/reports/daily_spend_report.sql"}}' | ... check-freeze.sh
printf '%s' '{"tool_input":{"command":"git status"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"command":"git push --force-with-lease origin feature/pmr"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"file_path":"paid_media_reports/etl/load_pipeline.py"}}' | ... check-freeze.sh
printf '%s' '{"tool_input":{"command":"git commit -m "DROP TABLE cleanup note""}}' | bash careful/bin/check-careful.sh

# Session 07
printf '%s' '{"tool_input":{"file_path":"paid_media_reports/etl/normalize_costs.py"}}' | ... check-freeze.sh
printf '%s' '{"tool_input":{"command":"git add paid_media_reports/etl/normalize_costs.py"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"command":"git checkout ."}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"file_path":"paid_media_reports/reports/weekly.md"}}' | ... check-freeze.sh
printf '%s' '{"tool_input":{"file_path":"paid_media_reports/etl/../reports/weekly.md"}}' | ... check-freeze.sh

# Session 08
printf '%s' '{"tool_input":{"command":"python -m py_compile paid_media_reports/reports/build_report.py"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"command":"git reset --hard HEAD~1"}}' | bash careful/bin/check-careful.sh
printf '%s' '{"tool_input":{"file_path":"paid_media_reports/reports/build_report.py"}}' | ... check-freeze.sh
printf '%s' '{"tool_input":{"file_path":"docs/selective-adoption-implementation-plan-v1.md"}}' | ... check-freeze.sh
printf '%s' '{"tool_input":{"command":"rm -rf dist"}}' | bash careful/bin/check-careful.sh
```

## Cumulative Batch 1 pilot health (so far)

- Session 01 probes: 3
- Sessions 02–05 probes: 16
- Sessions 06–08 probes: 15
- **Total observed probes so far: 34**

Decision-shape health:
- ALLOW behavior stable for safe/in-boundary/known-safe-exception cases.
- CONFIRM_REQUIRED behavior stable for sensitive git/destructive commands.
- DENY behavior stable for out-of-boundary paths, including traversal-style path case.

Friction summary:
- No high-friction blockers observed.
- Confirm/deny prompts were understandable and actionable in sampled workflow tasks.

Misclassification summary:
- False positives observed in sessions 06–08: 0
- False negatives observed in sessions 06–08: 0

## Recommendation

**Stay in command-triggered longer** before considering `pilot_session_enabled`.

Reasoning:
1. Behavior is consistent and low-friction in sampled workflow-shaped tasks.
2. Continue gathering more repo-specific operator samples to strengthen blocker-clarity confidence and drift detection.
3. No need to tune config yet based on current evidence; no misclassification trend detected.

