# Selective absorption analysis for mature agent ecosystem

Canonical module family: **Scoped Change Control** (historical label: Batch 1).


## Repository map (focused areas)

- `office-hours/`: planning-first ideation workflow with startup/builder modes and mandatory design-doc output.
- `autoplan/`: sequential auto-review pipeline that executes CEO → Design → Eng → DX reviews with decision logging.
- `review/`: pre-landing PR review + checklist execution + optional specialist sub-reviews.
- `careful/`, `freeze/`, `guard/`: runtime safety hooks for destructive command warnings and edit-boundary enforcement.
- `hosts/openclaw.ts`, `scripts/host-adapters/openclaw-adapter.ts`, `openclaw/*.md`: host adaptation layer for OpenClaw and orchestrator-facing pipeline injection docs.

## High-value components (what they do in practice)

### office-hours
- Forces pre-implementation product/design thinking and explicitly blocks implementation actions (design-doc only).
- Splits behavior into startup diagnostic mode vs builder ideation mode; asks goal first and routes depth/style from that.
- Requires premise confirmation and alternatives before recommendation.

Why it matters:
- High leverage for reducing premature coding and improving problem framing before architecture decisions harden.

### autoplan
- Runs full review gauntlet from existing review skills, sequentially, not in parallel.
- Auto-decides most intermediate choices using explicit decision principles; preserves user control for premise confirmation and “user challenge” decisions.
- Writes decision audit trail and restore point so edits are reversible and review provenance is visible.

Why it matters:
- Converts fragmented/manual planning review into an auditable, low-friction, high-throughput pipeline.

### review
- Executes diff-based pre-landing review against a checklist, including critical risk categories and fix-first workflow.
- Can auto-fix mechanical findings, ask for approval on judgment calls, and persist structured review outcome for downstream ship automation.
- Adds optional specialist dispatch for larger diffs and adversarial pass.

Why it matters:
- Strong quality gate that catches production-impacting classes often missed by tests alone.

### careful / freeze / guard
- `careful`: command-level destructive-pattern detector; warns and asks before risky operations.
- `freeze`: blocks writes/edits outside an allowed directory boundary.
- `guard`: composes both into one safety mode.

Why it matters:
- Simple, composable safety controls with low operational overhead.

### OpenClaw integration patterns
- Dedicated host config rewrites paths/tool-language and suppresses host-incompatible resolver sections.
- Adapter performs semantic rewrites (AskUserQuestion → direct user prompt, Agent tool language → sessions_spawn language, browse command adjustment).
- Orchestrator docs define routing tiers (SIMPLE/MEDIUM/HEAVY/FULL/PLAN) and enforce spawn-not-redirect behavior.

Why it matters:
- Clear portability pattern: keep core skill intent, then apply host-specific transforms and routing rules.

## ADOPT / ADAPT / IGNORE

| Area | Decision | Rationale | Minimal integration suggestion |
|---|---|---|---|
| `careful` destructive-command warning hook | **ADOPT** | Small, isolated, low-conflict safety primitive with immediate risk reduction. | Port pattern list + ask-before-run behavior into your global guard layer as a policy module. |
| `freeze` edit-boundary hook | **ADOPT** | Strong additive control for scoped edits, useful in incident/debug sessions. | Add as optional “write-scope lock” in Orion governance policy; keep current command names, only import logic. |
| `guard` composite mode | **ADOPT** | Pure composition of two primitives; easy win. | Implement as alias/policy profile in your existing safety mode taxonomy. |
| `review` fix-first + checklist discipline | **ADAPT** | High value, but current implementation is tightly coupled to gstack paths and conventions. | Extract the pattern (critical categories + auto-fix/ask split + persisted review ledger) into your existing review skill names and storage. |
| `autoplan` decision framework + audit trail | **ADAPT** | Useful orchestration idea, but it imposes gstack phase names, principles language, and doc sections. | Re-map phases to OpenClaw/Orion naming; keep only: sequential pipeline, decision classes (mechanical/taste/user-challenge), restore point, audit log. |
| `office-hours` pre-build forcing function | **ADAPT** | Strong planning behavior, but tone/method is opinionated and startup-heavy by default. | Keep your current skill names; add lightweight “premises + alternatives + no-code gate” subroutine to project-level planning entrypoint. |
| OpenClaw host adapter + host config approach | **ADAPT** | Architecture is valuable; string mappings and suppressed resolvers are gstack-specific. | Copy adapter pattern, not exact mappings. Build transformation layer keyed to your OpenClaw + Orion policy vocabulary. |
| OpenClaw injected FULL pipeline (`autoplan -> implement -> ship`) | **IGNORE** (as-is) | Too prescriptive for mature systems with existing governance and stable workflow responsibilities. | Do not import full pipeline; at most borrow tiered dispatch concept under current orchestrator routes. |
| gstack-specific telemetry/preamble/upgrade prompts | **IGNORE** | Adds parallel operational surface and conflicts with established governance/telemetry controls. | Keep your existing telemetry/governance stack; do not import gstack runtime preamble machinery. |

## Conflict risks (for existing mature stack)

1. **Skill system collision**
   - gstack assumes proactive skill auto-invocation and its own routing section format; this can conflict with established project/global Claude skill routers.

2. **Naming convention drift**
   - Commands like `/office-hours`, `/autoplan`, `/review`, `/guard` can create duplicate responsibility names vs existing OpenClaw/Orion nomenclature.

3. **Governance/kernel overlap**
   - gstack introduces independent safety, telemetry, and policy prompts that may bypass or duplicate Orion governance controls.

4. **Workflow responsibility duplication**
   - `autoplan` and orchestrator “full pipeline” docs bundle planning/review/implementation/ship into one macro flow, which may overlap existing OpenClaw workflow layers.

## Minimal integration plan (compatibility-first)

1. **Week 1: absorb safety primitives only**
   - Implement `careful` + `freeze` logic as Orion-native policy modules (no new user-facing command names required).
   - Add a single composite profile equivalent to `guard`.

2. **Week 2: absorb review pattern, not review identity**
   - Introduce fix-first decisioning (AUTO-FIX vs ASK) and a compact persistent review ledger in your existing review workflow.
   - Add only the highest-signal checklist categories first (SQL/data safety, trust boundary, race conditions, shell injection, enum completeness).

3. **Week 3: absorb planning discipline incrementally**
   - Add a “no implementation until premises + alternatives are explicit” gate to your existing planning entrypoint.
   - Add `autoplan`-style decision classification (mechanical/taste/user-challenge) to existing OpenClaw planning runs, but keep current phase names and governance hooks.

4. **Do not migrate wholesale**
   - No gstack routing section injection.
   - No gstack telemetry/upgrade/preamble runtime.
   - No replacement of existing OpenClaw/Orion command hierarchy.

## Priority order (highest leverage per integration risk)

1. **Safety hooks (`careful`/`freeze`/`guard`)** — highest ROI, lowest architecture disruption.
2. **Review fix-first mechanics + minimal critical checklist** — high quality gain, moderate adaptation effort.
3. **Planning gates from `office-hours` + decision taxonomy from `autoplan`** — medium-high leverage, requires naming/governance alignment.
4. **OpenClaw host-adapter architecture pattern** — valuable for long-term portability, but lower immediate user-visible impact.
