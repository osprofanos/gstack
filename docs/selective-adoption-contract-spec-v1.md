# Orion Selective Adoption Contract Spec v1 (No-Code)

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


Status: Draft v1 (implementation-ready contract)  
Scope: `command_safety_gate`, `edit_scope_boundary`, `change_guard_profile`, `premerge_risk_review`, `review_action_router`, `review_event_ledger`

## 0) Contract posture

- Conservative by default: ambiguous risk => block or require explicit confirmation.
- No new layers: contracts bind existing Orion interceptors, Orion policies, OpenClaw workflow stages, and current execution pathways.
- Additive only: existing governance arbitration remains authoritative.

---

## 1) Component contracts

## 1.1 `command_safety_gate`

**Purpose**
- Pre-execution policy decision for shell/exec operations.

**Input contract (minimum)**
```json
{
  "request_id": "uuid",
  "session_id": "string",
  "repo": "slug",
  "branch": "string",
  "actor": "user|agent|workflow",
  "command": "string",
  "context": {
    "profile": "string|null",
    "workflow_stage": "string|null",
    "project_override": false
  }
}
```

**Output decision contract**
```json
{
  "component": "command_safety_gate",
  "decision": "allow|confirm_required|deny",
  "risk_class": "none|destructive_fs|destructive_db|history_rewrite|infra_destructive|unknown_high_risk",
  "confidence": 0.0,
  "reason": "short text",
  "override_allowed": true,
  "evidence": ["pattern_id"],
  "ttl_seconds": 0
}
```

**Required behavior**
- Known destructive patterns: `confirm_required` (or `deny` if override forbidden by active policy).
- Parse failure + suspicious command shape: `deny` with `risk_class=unknown_high_risk`.
- Empty/no-op command: `allow`.

---

## 1.2 `edit_scope_boundary`

**Purpose**
- Pre-write/edit path boundary decision.

**Input contract (minimum)**
```json
{
  "request_id": "uuid",
  "session_id": "string",
  "operation": "write|edit|patch",
  "target_path": "string",
  "context": {
    "active_boundary": "/abs/path/or/null",
    "profile": "string|null",
    "project_override": false
  }
}
```

**Output decision contract**
```json
{
  "component": "edit_scope_boundary",
  "decision": "allow|deny",
  "normalized_target": "/abs/path",
  "normalized_boundary": "/abs/path/or/null",
  "reason": "inside_boundary|outside_boundary|no_boundary|path_unresolvable",
  "override_allowed": false
}
```

**Required behavior**
- Active boundary + unresolved path => `deny`.
- Active boundary + outside target => `deny`.
- No active boundary => `allow` (unless stricter project override says otherwise).

---

## 1.3 `change_guard_profile`

**Purpose**
- Session profile contract that composes `command_safety_gate` + `edit_scope_boundary`.

**State contract**
```json
{
  "profile": "change_guard_profile",
  "active": true,
  "session_id": "string",
  "boundary_root": "/abs/path/or/null",
  "rules": {
    "command_safety_gate": "enabled",
    "edit_scope_boundary": "enabled"
  },
  "override_policy": "kernel_controlled"
}
```

**Required behavior**
- Profile activation is atomic: both policies enabled together.
- Boundary not set => command safety still active; boundary policy returns `no_boundary` behavior.

---

## 1.4 `premerge_risk_review`

**Purpose**
- Required pre-merge stage producing normalized risk findings.

**Input contract (minimum)**
```json
{
  "run_id": "uuid",
  "repo": "slug",
  "branch": "string",
  "base_ref": "origin/main",
  "diff_ref": "string",
  "categories": [
    "sql_safety",
    "race_condition",
    "trust_boundary",
    "shell_injection",
    "enum_completeness"
  ]
}
```

**Output contract**
```json
{
  "component": "premerge_risk_review",
  "status": "clean|issues_found|failed",
  "findings": [],
  "summary": {"critical": 0, "high": 0, "medium": 0, "info": 0},
  "failure_mode": "none|diff_unavailable|base_unavailable|analysis_error"
}
```

**Required behavior**
- If diff/base unavailable => `failed` (fail-closed for merge gating).
- Enum completeness may read out-of-diff references in bounded scope.

---

## 1.5 `review_action_router`

**Purpose**
- Deterministic routing for each finding: `auto_fix_candidate|ask_user|defer`.

**Input contract (minimum)**
```json
{
  "run_id": "uuid",
  "finding": {
    "finding_id": "string",
    "category": "string",
    "severity": "critical|high|medium|info",
    "confidence": 0.0,
    "summary": "string"
  },
  "context": {
    "workflow_stage": "premerge_risk_review",
    "project_override": false
  }
}
```

**Output decision contract**
```json
{
  "component": "review_action_router",
  "finding_id": "string",
  "route": "auto_fix_candidate|ask_user|defer",
  "reason_code": "critical_default|mechanical_safe|low_confidence|policy_override",
  "requires_user_confirmation": true
}
```

**Required behavior**
- Critical severity defaults to `ask_user` unless explicitly marked mechanical+reversible by policy.
- Low confidence (< configured threshold) => `ask_user` or `defer` (never auto-fix).

---

## 1.6 `review_event_ledger`

**Purpose**
- Append-only contract for review/audit events.

**Event contract**
```json
{
  "event_id": "uuid",
  "event_ts": "RFC3339",
  "event_type": "review_run|finding_routed|finding_applied|finding_skipped|policy_intercept",
  "repo": "slug",
  "branch": "string",
  "run_id": "uuid",
  "payload": {},
  "integrity": {"schema_version": "v1", "hash": "optional"}
}
```

**Required behavior**
- Append-only writes.
- Write failure in pre-merge context => fail-closed (`status=failed`) unless kernel override explicitly permits degraded mode.

---

## 2) Arbitration order

Authoritative order for write/exec + review flow:

1. **Orion kernel interceptors** (existing approvals/denials; highest precedence)
2. **Orion safety policies**
   - `command_safety_gate` for exec
   - `edit_scope_boundary` for write/edit
3. **Active profile enforcement** (`change_guard_profile` toggles policy enablement/state)
4. **OpenClaw workflow stage** (`premerge_risk_review` before merge path)
5. **Routing stage** (`review_action_router` per finding)
6. **Execution/apply path** (existing implementation flow)
7. **Ledger append** (`review_event_ledger`)

Rule: earlier denial is terminal unless explicit kernel-approved override exists.

---

## 3) Conflict resolution rules

1. **Kernel > everything**
   - If kernel says `deny`, terminal deny.
2. **Deny beats allow**
   - If one active policy denies and another allows, final = deny.
3. **Confirm beats allow**
   - If no denies and any policy returns `confirm_required`, final = confirmation gate.
4. **Project override cannot weaken kernel**
   - Project overrides may tighten behavior, not relax kernel denials.
5. **Workflow failure is fail-closed for pre-merge**
   - `premerge_risk_review=status=failed` => merge blocked.
6. **Ledger write failures**
   - Pre-merge: fail-closed unless explicit kernel degraded-mode override.
   - Non-merge contexts: return warning + continue if allowed by policy.

---

## 4) Activation truth tables

## 4.1 `command_safety_gate`

| Always-on | Profile active | Command-triggered | Project override | Effective state |
|---|---|---|---|---|
| T | * | * | * | Enabled |
| F | T | * | * | Enabled |
| F | F | T | * | Enabled (session) |
| F | F | F | T (enable) | Enabled |
| F | F | F | F | Disabled |

## 4.2 `edit_scope_boundary`

| Always-on | Profile active | Command-triggered boundary | Project override | Effective state |
|---|---|---|---|---|
| T | * | * | * | Enabled (requires boundary source) |
| F | T | * | * | Enabled |
| F | F | T | * | Enabled |
| F | F | F | T (require) | Enabled (deny until boundary set) |
| F | F | F | F | Disabled |

## 4.3 `change_guard_profile`

| Always-on | Profile active | Command-triggered | Project override | Effective state |
|---|---|---|---|---|
| * | T | * | * | Enabled |
| * | F | T | * | Enabled |
| * | F | F | T (default profile) | Enabled |
| * | F | F | F | Disabled |

## 4.4 `premerge_risk_review`

| Always-on | Profile active | Command-triggered | Project override | Effective state |
|---|---|---|---|---|
| T | * | * | * | Required |
| F | * | T | * | Required (run scope) |
| F | * | F | T (require) | Required |
| F | * | F | F | Optional/Disabled |

## 4.5 `review_action_router`

| Always-on | Profile active | Command-triggered | Project override | Effective state |
|---|---|---|---|---|
| T | * | * | * | Enabled |
| F | * | * | T (enable) | Enabled |
| F | * | * | F | Enabled only when review stage emits findings |

## 4.6 `review_event_ledger`

| Always-on | Profile active | Command-triggered | Project override | Effective state |
|---|---|---|---|---|
| T | * | * | * | Enabled |
| F | * | * | T (require) | Enabled |
| F | * | * | F | Enabled for pre-merge runs; optional elsewhere |

---

## 5) Minimal JSON examples

## 5.1 Safety decision
```json
{
  "component": "command_safety_gate",
  "decision": "confirm_required",
  "risk_class": "history_rewrite",
  "confidence": 0.98,
  "reason": "force push detected",
  "override_allowed": true,
  "evidence": ["git_force_push"],
  "ttl_seconds": 300
}
```

## 5.2 Boundary decision
```json
{
  "component": "edit_scope_boundary",
  "decision": "deny",
  "normalized_target": "/repo/src/other/file.ts",
  "normalized_boundary": "/repo/src/module-a",
  "reason": "outside_boundary",
  "override_allowed": false
}
```

## 5.3 Review finding
```json
{
  "finding_id": "f_9f2c",
  "category": "sql_safety",
  "severity": "critical",
  "confidence": 0.91,
  "location": {"path": "app/models/user.rb", "line": 42},
  "summary": "string interpolation in query",
  "recommended_action": "ask_user"
}
```

## 5.4 Routing decision
```json
{
  "component": "review_action_router",
  "finding_id": "f_9f2c",
  "route": "ask_user",
  "reason_code": "critical_default",
  "requires_user_confirmation": true
}
```

## 5.5 Ledger event
```json
{
  "event_id": "e_123",
  "event_ts": "2026-04-20T10:30:00Z",
  "event_type": "finding_routed",
  "repo": "payments-api",
  "branch": "feature/safe-query",
  "run_id": "run_456",
  "payload": {"finding_id": "f_9f2c", "route": "ask_user"},
  "integrity": {"schema_version": "v1"}
}
```

---

## 6) Edge cases and failure behavior

1. **Command parse failure**
   - Behavior: `deny` with `unknown_high_risk`.
2. **Boundary path cannot normalize**
   - Behavior: `deny` (`path_unresolvable`).
3. **Boundary not set but boundary required by override**
   - Behavior: `deny` until set.
4. **Base branch unavailable for review**
   - Behavior: `premerge_risk_review=failed`; pre-merge blocked.
5. **Diff too large/time budget exceeded**
   - Behavior: `failed` unless policy allows segmented review mode; still block merge if unresolved.
6. **Router receives malformed finding**
   - Behavior: route=`defer`, emit ledger warning event, block auto-fix.
7. **Ledger write failure**
   - Pre-merge: fail-closed.
   - Non-pre-merge: warn and continue only if kernel policy permits.
8. **Conflicting overrides (global enable vs project disable)**
   - Behavior: stricter rule wins; if equal strictness conflict, kernel default wins.

---

## 7) Naming warnings (future growth)

1. `command_safety_gate`
   - Risk: may outgrow shell-only scope if later includes API/db mutation guards.
   - Watch trigger: scope expands beyond command execution.

2. `edit_scope_boundary`
   - Risk: name may be narrow if later extended to artifact generation, move/rename, or non-file resources.
   - Watch trigger: boundary expands beyond write/edit file paths.

3. `change_guard_profile`
   - Risk: “change” may under-specify production/runtime safeguards if profile expands.
   - Watch trigger: profile adds deploy/runtime controls.

4. `premerge_risk_review`
   - Risk: name tied to pre-merge only; may not fit reuse in pre-commit or post-merge audits.
   - Watch trigger: stage reused outside merge gating.

5. `review_action_router`
   - Risk: could imply broad workflow orchestration beyond finding routing.
   - Watch trigger: starts handling batching/escalation/assignment.

6. `review_event_ledger`
   - Risk: if expanded into analytics/BI, “ledger” may mislead expectations.
   - Watch trigger: introduces aggregates, dashboards, trend pipelines.

