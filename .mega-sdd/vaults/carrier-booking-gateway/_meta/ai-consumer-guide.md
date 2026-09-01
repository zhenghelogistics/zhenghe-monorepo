# AI Consumer Guide — mega-sdd vault protocol

> **Static copy installed by mega-sdd; identical across vaults; do not hand-edit — re-copied on regen** (a plain `cp` of this template at vault generation time). Per-vault specifics (which P1 OQ clusters block which work areas, layer-routing anchors, vault metadata) live in `vault.md` (legacy: 00-index.md §Implementation Notes) — this file carries the GENERIC consumer protocol only.

This guide is for AI dev tools (Claude Code, Cursor, etc.) and humans that read the vault as source of truth when writing/modifying code. The vault (layout-2: `vault.md` + `model.md` + `flows.md` + `constraints.md`; legacy: `00-index.md` … `06-constraints.md`) is the single source of truth for requirements; this file tells you how to consume it safely.

## MANDATORY before writing/modifying any code

1. **Confirm project shape & mode with the user**:
   - Ask: *"This vault states shape `<shape>` and mode `<mode>` (see the vault.md frontmatter lock; legacy: `00-index.md` §Vault Lock Status). Are you working in a project that matches?"*
   - On mismatch → STOP, escalate.

2. **For mode `existing`** — additional MANDATORY steps:
   - Ask the user: *"Share a short description of the existing codebase (project root, framework, key tables that are relevant), or confirm I should scan first before continuing."*
   - **Cross-check entities** ([[03-data-model]]) against the existing schema:
     - New entity in vault, name doesn't collide with existing → safe to create.
     - Vault entity that shares a name with an existing one → STOP, clarify extend vs replace.
   - **Cross-check flows** ([[04-flows]]) against existing routes/handlers/cron jobs:
     - New flow, no collision → safe to add.
     - Flow that touches an existing endpoint/job → STOP, clarify extend vs replace.
   - **Cross-check decisions** ([[05-decisions]]) against existing patterns:
     - Decision that **conflicts** with an existing pattern → STOP, escalate to architect for a transition plan.

3. **For mode `new`** — checks still apply:
   - Confirm tech stack from the vault with the user ([[02-architecture]] may still have Open Questions on stack).
   - If P1 Open Questions are unresolved → STOP, do not auto-pick a stack default.

4. **Use the relevant layer section based on what you're implementing**:
   - Working on backend → focus on [[02-architecture#Backend]] + the backend section of [[04-flows]].
   - Working on UI (mobile/web) → focus on the relevant UI layer in [[02-architecture]] + user flows in [[04-flows]].
   - Cross-cutting feature → check the cross-cutting flows section + multiple layer sections.

## During implementation

- **Do not inject requirements** that aren't in the vault. If a new requirement is needed → STOP, append it to the vault's `## Open Questions` (layout-2: constraints.md, with an `[origin:]` token; legacy: the relevant doc) and ask the user.
- **Do not skip Definition of Done**. For each flow you implement, validate DoD before marking it complete.
- **Cite the vault** in commit messages when touching business logic — e.g., `feat: cap tenor per vault flows.md F-U-001 step 5`. Never put a vault claim/flow/OQ id in a CODE COMMENT — those id strings rot into misinformation and no validator consumes them there; trace lives in commits, unit specs, and reports.

## When you encounter an inconsistency

- Vault internal conflict (e.g., doc 03 vs doc 04) → STOP, surface to the user with quotes from both sides.
- Vault vs existing code conflict → STOP, escalate to user. Show the vault quote + the existing-code reference.
- Vault vs original PRD (if user grants PRD access) → STOP, escalate to user. The vault should reflect the PRD; if not, the vault is stale.

## Halt protocol for autonomous runs

In **interactive mode** (chat with a human), "STOP and ask user" works fine — surface the issue in chat and wait. In **autonomous mode** (agent runners, CI tasks, headless workflows, the `flow` orchestrator), silent halt loses the signal. Instead, emit a structured `blocker` artifact so the runner can route it.

The unified envelope (per the mega-sdd plugin's `references/halt-protocol.md` §halt-protocol) covers three blocker types: `oq_blocker` (unresolved P1 OQ), `diff_conflict` (diff-vault conflict), and `drift_framework_mismatch` (detect-drift framework mismatch).

When you hit an unresolved P1 OQ that blocks your current task, emit (in addition to any chat response):

```yaml
blocker:
  type: oq_blocker
  tag: OQ-AR-1
  priority: P1
  context: "Implementing F-U-001 backend"
  resolver_owner: "Mike Patel (Eng Lead)"
  resolver_route: "ask in PM Slack channel #timeoff-team or via PRD §L review"
  vault_version: "1.0"
  source_skill: generate-intent
```

For multiple blockers in one task, emit an array:

```yaml
blockers:
  - type: oq_blocker
    tag: OQ-AR-1
    priority: P1
    context: "Implementing F-U-001 backend"
    resolver_owner: "Mike Patel"
    resolver_route: "ask in #timeoff-team"
    vault_version: "1.0"
    source_skill: generate-intent
  - type: oq_blocker
    tag: OQ-DM-1
    priority: P1
    context: "Implementing F-U-001 backend"
    resolver_owner: "Mike Patel + security"
    resolver_route: "ask in #timeoff-team"
    vault_version: "1.0"
    source_skill: generate-intent
```

The agent runner decides what to do (page resolver, create ticket, post to Slack). The skill's job is to emit the structured artifact reliably — don't paraphrase, don't drop fields. See the mega-sdd plugin's `references/halt-protocol.md` §halt-protocol for the full schema and the two non-OQ types (`diff_conflict`, `drift_framework_mismatch`).

> **Backward compat note**: older vaults may emit the bare `oq_blocker:` (legacy form). AI consumers should accept both `oq_blocker:` and `blocker: type: oq_blocker` shapes.

## Parallel-work guidance while P1s are unresolved

If your task is fully blocked by P1 OQs but you want to make incremental progress, work on artifacts that don't depend on the unresolved decisions:

- **From DoD bullets** (in [[04-flows]]): draft test specs / Gherkin scenarios. The DoD is the test contract.
- **From entities** (in [[03-data-model]]): scaffold ORM models / type definitions with `// TODO(OQ-...): resolved type pending` markers.
- **From flows**: sketch UI mocks / API stub signatures using vault-stated names but no business logic yet.
- **From OOS section**: confirm with PM what's NOT in scope — saves wasted scaffolding.

Mark each artifact with the OQ tag(s) it depends on so it can be revisited once resolved. Keep these artifacts in a `WIP/` or feature-flagged path so they don't accidentally ship.

## Companion skills for vault evolution

When the user needs to update or reconcile the vault, route them to the appropriate companion skill instead of editing the vault free-hand:

- **Stakeholder OQ resolution round** → `resolve-oq`. Walks the OQ roll-up by priority, captures answers, marks resolved entries with stable tags + bumps vault version + Changelog.
- **PRD/BRD source revised** → `diff-vault`. Computes structured diff between the new source and current vault state. Surfaces conflicts (Resolved-OQ vs new PRD) for explicit user decision; never silently overwrites.
- **Codebase reconciliation (`Implementation mode: existing` only)** → `detect-drift`. Scans the live codebase, flags entity/flow/decision drift between the vault target and current code reality. Produces `DRIFT-REPORT.md` for review.

These skills share the vault as state. They preserve OQ tag identity, ADR `D-XXX` numbering, and Changelog history across rounds — running them is the safe way to evolve the vault. Direct edits are still allowed but must follow the same conventions (preserve identifiers, append to Changelog, bump version).

## Standard terms

Generic cross-doc terms and acronyms (product-specific PRD terms live in `vault.md ## Glossary`; legacy: 00-index.md):

| Term | Definition |
|------|----------|
| ADR | Architecture Decision Record — record of a technical decision with context, decision, consequences |
| DBML | Database Markup Language — text format for defining database schema |
| DoD | Definition of Done — observable criteria that mark a flow/task complete |
| FK | Foreign Key |
| NFR | Non-Functional Requirement — performance, availability, security, etc. |
| OQ | Open Question — ambiguity that needs a stakeholder answer |
| RTO | Recovery Time Objective |
| RPO | Recovery Point Objective |
| SLO | Service Level Objective |
| design tokens (cond.) | Named design values (color, typography, spacing) shared across components. Source-mirrored from Figma variables / tokens.json / PRD. |
| design system (cond.) | Set of components + tokens + a11y + voice rules that constrain UI implementation. |
| WCAG (cond.) | Web Content Accessibility Guidelines — international standard for a11y. Vault uses a level only if source explicitly states it. |
| a11y (cond.) | Numeronym for "accessibility" (a + 11 letters + y). |
| semantic HTML (cond.) | Use of meaningful HTML elements (`<button>`, `<nav>`, `<main>`, etc.) for accessibility and structure. |

> Rows marked `(cond.)` are relevant only when the vault carries design-system sections (`02-architecture#UI components & patterns` / `06-constraints#Design System`); ignore them otherwise. This table is static — it never varies per vault.
