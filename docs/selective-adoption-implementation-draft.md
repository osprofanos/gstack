# Selective Adoption Implementation Draft (No-Code)

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


## Design constraints (binding)

- Orion governance remains the source of truth (policy arbitration, approvals, audit).
- OpenClaw remains orchestration/runtime entrypoint.
- Project/global Claude skills remain intact; no duplicate parallel skill families.
- New components are additive overlays and can be disabled cleanly.
- Stable atemporal naming: no imported gstack command names.

---

## Proposed native names

## Phase 1 (safety primitives)

1. **Orion Policy: `command_safety_gate`**
   - Careful-equivalent behavior.
2. **Orion Policy: `edit_scope_boundary`**
   - Freeze-equivalent behavior.
3. **Orion Profile: `change_guard_profile`**
   - Guard-equivalent composition of `command_safety_gate` + `edit_scope_boundary`.

## Phase 2 (review discipline)

4. **OpenClaw Workflow Step: `premerge_risk_review`**
   - Review-equivalent pass for PR/diff risk checks.
5. **Orion Decision Policy: `review_action_router`**
   - AUTO-FIX vs ASK routing policy.
6. **Orion Audit Artifact: `review_event_ledger`**
   - Compact durable ledger of review outcomes/actions.

Naming rationale:
- Descriptive and governance-native (policy/profile/workflow/artifact).
- Avoids gstack terms and avoids creating a second naming hierarchy.

---

## Artifact placement map

| Artifact | Layer | Why here |
|---|---|---|
| `command_safety_gate` | **Global** (Orion kernel policy registry) | Cross-project command-risk behavior should be consistent and centrally governed. |
| `edit_scope_boundary` | **Global** (Orion kernel policy registry) | Core write-boundary enforcement belongs in governance, not per-repo logic. |
| `change_guard_profile` | **OpenClaw/Core** profile catalog + Orion policy references | Operationally invoked by workflow/operator; composes global policies without duplicating them. |
| `premerge_risk_review` | **OpenClaw/Core** workflow stage | Naturally belongs in pre-merge pipeline orchestration. |
| `review_action_router` | **Global** (Orion decision policy) | Keeps fix-vs-ask decisions standardized across projects. |
| `review_event_ledger` schema + writer contract | **Global schema**, **Per-project storage** | Global consistency + local project auditability and retention control. |
| Optional stricter overrides (extra patterns/checks) | **Per-project optional layer** | Allows regulated/high-risk repos to extend policy without forking core behavior. |

---

## Responsibilities and boundaries

## 1) `command_safety_gate` (careful-equivalent)

**Responsibilities**
- Inspect proposed shell/exec commands pre-execution.
- Detect destructive/high-impact patterns (recursive delete, forced history rewrite, destructive DB ops, destructive infra ops).
- Return an **intercept decision**: `allow` or `confirm_required`.

**Boundaries**
- Does not execute commands.
- Does not decide project-specific exceptions unless provided via per-project allowlist extension.
- Does not replace Orion approval escalation; it feeds it.

## 2) `edit_scope_boundary` (freeze-equivalent)

**Responsibilities**
- Enforce write/edit paths against an active allowed-root boundary.
- Return `allow` or `deny` pre-write.
- Normalize paths (absolute resolution + symlink-aware checks).

**Boundaries**
- Applies to edit/write operations only (not read).
- Not a security sandbox replacement; governance control only.
- Boundary lifecycle handled by activation context/profile (not by ad-hoc tool logic).

## 3) `change_guard_profile` (guard-equivalent)

**Responsibilities**
- Activate both `command_safety_gate` and `edit_scope_boundary` under one operator intent.
- Publish session-scoped state (active profile, boundary root, override counters).

**Boundaries**
- No additional rules beyond composition unless explicitly configured.
- Never bypasses Orion kernel policy precedence.

## 4) `premerge_risk_review` (review-equivalent)

**Responsibilities**
- Run against diff vs base.
- Apply **critical subset only** in v1:
  1. SQL/data safety
  2. Race/concurrency risk
  3. Trust-boundary / untrusted-output handling
  4. Shell/command injection
  5. Enum/value completeness (including targeted out-of-diff references)
- Produce findings with severity and confidence.

**Boundaries**
- Not a full code-quality replacement.
- No broad style policing in v1.
- No auto-commit/ship actions.

## 5) `review_action_router` (AUTO-FIX vs ASK)

**Responsibilities**
- Classify each finding action as:
  - `auto_fix_candidate`
  - `ask_user`
  - `defer`
- Apply conservative defaults: critical findings bias to `ask_user` unless mechanically safe and deterministic.

**Boundaries**
- Classification policy only; does not apply patches itself.
- Any fix application remains inside existing OpenClaw implementation flow.

## 6) `review_event_ledger`

**Responsibilities**
- Append compact structured events for review runs and per-finding actions.
- Enable branch-level audit trail and governance observability.

**Boundaries**
- Minimal schema; not a full analytics platform.
- PII/code-content minimization by design (metadata + fingerprints, not large blobs).

---

## Activation model

| Component | Activation |
|---|---|
| `command_safety_gate` | **Profile-based default ON** in risky contexts (prod/hotfix/infrastructure workflows), otherwise command-triggered via safety profile. |
| `edit_scope_boundary` | **Command-triggered** (operator sets allowed path) or included by profile. |
| `change_guard_profile` | **Command-triggered profile** for high-safety sessions; can be set as default in protected repos. |
| `premerge_risk_review` | **Always-on** for pre-merge workflow stage in OpenClaw pipelines. |
| `review_action_router` | **Always-on** when `premerge_risk_review` emits findings. |
| `review_event_ledger` | **Always-on write** for each review run and finding action. |

Activation principle:
- Keep friction low in normal dev; enforce stronger controls in clearly risky modes and pre-merge gates.

---

## Minimal interfaces and output formats

## A) Safety gate decision interface

```json
{
  "policy": "command_safety_gate",
  "decision": "allow | confirm_required",
  "risk_class": "none | destructive_fs | destructive_db | history_rewrite | infra_destructive",
  "reason": "short human-readable explanation",
  "override_allowed": true,
  "evidence": ["matched_pattern_id"]
}
```

## B) Edit boundary decision interface

```json
{
  "policy": "edit_scope_boundary",
  "decision": "allow | deny",
  "allowed_root": "/abs/path/root/",
  "target_path": "/abs/path/root/or/file",
  "reason": "outside_boundary | no_boundary | inside_boundary"
}
```

## C) Review finding (compact)

```json
{
  "finding_id": "hash(path:line:category)",
  "category": "sql_safety|race_condition|trust_boundary|shell_injection|enum_completeness",
  "severity": "critical|high|medium|info",
  "confidence": 0.0,
  "location": {"path": "...", "line": 0},
  "summary": "one-line issue statement",
  "recommended_action": "auto_fix_candidate|ask_user|defer"
}
```

## D) Compact review ledger event

```json
{
  "event_ts": "2026-04-20T00:00:00Z",
  "repo": "project-slug",
  "branch": "feature/x",
  "workflow": "premerge_risk_review",
  "base_ref": "origin/main",
  "run_id": "uuid",
  "status": "clean|issues_found",
  "totals": {"critical": 0, "high": 0, "medium": 0, "info": 0},
  "actions": {"auto_fixed": 0, "asked": 0, "deferred": 0},
  "findings": [
    {"finding_id": "...", "severity": "critical", "action": "ask_user"}
  ]
}
```

Storage guidance:
- Append-only JSONL per project (e.g., `.orion/audit/review-events.jsonl`) with optional global index pointers.

---

## Risks and edge cases

1. **Policy collision risk**
   - Existing Orion approvals may already intercept destructive operations.
   - Mitigation: enforce single arbitration order (Orion kernel first), then additive policy checks, then workflow-specific prompts.

2. **Overblocking developer flow**
   - Aggressive patterns or strict boundaries can slow normal iteration.
   - Mitigation: profile-based defaults, safe allowlist for common build artifacts, explicit temporary override with audit event.

3. **Boundary bypass vectors**
   - Non-standard write paths/symlink behavior may bypass naive prefix checks.
   - Mitigation: normalized absolute path + symlink-aware resolution and explicit unresolved-path fail behavior.

4. **AUTO-FIX false confidence**
   - Misclassified critical findings could auto-apply risky changes.
   - Mitigation: conservative router defaults (`critical` => ask unless strictly mechanical and reversible).

5. **Ledger bloat / sensitivity**
   - Overly verbose records can accumulate and leak context.
   - Mitigation: compact schema, finding fingerprints, retention policy, avoid raw secret/code payloads.

6. **Duplicate review ownership**
   - Existing review skills may overlap new `premerge_risk_review` behavior.
   - Mitigation: position as a **single shared pre-merge stage** consumed by existing workflows, not a second reviewer.

---

## Phased rollout plan

## Phase 0 (prep, 1 week)
- Finalize names, schemas, and arbitration order with Orion/OpenClaw maintainers.
- Define compatibility contract with existing project/global skills.
- Pick 2 pilot repositories (one standard, one high-risk).

## Phase 1 (safety primitives, 1–2 weeks)
- Introduce `command_safety_gate` and `edit_scope_boundary` in Orion policy registry.
- Expose `change_guard_profile` in OpenClaw/core profile catalog.
- Launch as opt-in command/profile first; collect friction metrics and override rates.

Exit criteria:
- No governance conflicts.
- Acceptable false-positive rate.
- Operator confirmation UX validated.

## Phase 2 (review-equivalent core, 2 weeks)
- Add `premerge_risk_review` stage with critical subset only.
- Enable `review_action_router` with conservative defaults.
- Enable `review_event_ledger` JSONL writes per project.

Exit criteria:
- Stable findings taxonomy.
- Clear action split behavior (AUTO-FIX candidate vs ASK).
- Ledger useful for audit without excess noise.

## Phase 3 (hardening + selective expansion, 1–2 weeks)
- Add per-project optional overrides (extra forbidden patterns, stricter review thresholds).
- Tune confidence thresholds and action-router rules from pilot data.
- Promote from opt-in to default in target workflow profiles where justified.

Non-goals (explicit):
- No replacement of Orion governance.
- No import of gstack naming/preamble/telemetry machinery.
- No parallel second review system.
