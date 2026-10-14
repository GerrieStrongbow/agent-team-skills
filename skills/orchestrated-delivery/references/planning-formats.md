# Planning pipeline formats

Templates for every document type in `planning/`. Each subdirectory gets a README carrying its format so future agents (and future sessions) don't drift. Keep documents modular: one PRD per page/feature, one ADR per decision, one ticket per ticket branch.

## PRD (`planning/prds/<feature>.md`)

User-facing behaviour only — no tech stack, no implementation detail. PRDs are the source of truth over any app being cloned or replaced ("features may deliberately deviate from the original").

```markdown
# <Feature name>

Status: draft | approved (owner sign-off YYYY-MM-DD)
Priority: v1 | backlog

## Purpose — one paragraph, the job this page does for the user

## Requirements — numbered functional requirements (R1, R2, …)

## Deviations — where this intentionally differs from <the system it replaces>, and why

## Out of scope — explicit non-goals

## Acceptance criteria — testable, numbered (AC1, AC2, …)
```

R- and AC-numbers are referenced by specs and tickets — **never renumber**; mark removed items `(dropped)`. When the owner amends behaviour later, amend the PRD in the same commit as the ruling.

## ADR (`planning/adrs/NNNN-slug.md`)

Only for consequential, hard-to-reverse choices (aggregator selection, stack, auth strategy, derive-vs-materialise). Never deleted — superseded ADRs get `Status: superseded by NNNN`.

```markdown
# NNNN: <Decision title>

Status: proposed | accepted | superseded by NNNN
Date: YYYY-MM-DD

## Context — the problem and constraints

## Decision — what was chosen

## Alternatives — what was rejected and why

## Consequences — what this commits us to
```

Record live evidence in ADRs/specs when a decision rests on it (endpoint URLs observed, values that did or did not reconcile) — the next agent shouldn't have to rediscover it.

## Technical spec (`planning/specs/<feature>.md`)

Consumed by an implementer that follows it literally. Be explicit: exact file paths, data shapes, function signatures, SQL shapes, error handling. State which PRD requirements each section implements ("implements accounts R1–R4"). Include:

- Type/interface definitions verbatim (the implementer copies them).
- Performance invariants with the reasoning ("one round-trip per concern — never twelve per-period queries; serverless-driver latency lesson from ticket NNN").
- UI conventions by reference to the design system, plus the specific class treatments that are rulings, not suggestions.
- A Verification section: the fixture protocol, what evidence proves each AC.

Version-sensitive choices (library majors, APIs) are verified against current docs at write time and pinned with the date checked — never recalled from training data.

## Ticket (`planning/delivery/tickets/NNN-slug.md`)

One ticket = one implementer = one worktree = one ticket branch, squash-merged into the lead's feature branch as one commit. Number tickets globally (001…) so branch names, worktrees, and fixture prefixes all derive from the number.

```markdown
# NNN: <Title>

Status: todo | in progress | review | done ← lead sets done post-merge
Branch: <prefix>/NNN-slug, off <feature-branch>
Spec: ../../specs/<feature>.md; PRD: ../../prds/<feature>.md
Blocked by: NNN, NNN — or —

## Scope

Numbered, concrete build steps with file paths. End with the ownership
statement: "Owned files: X, Y, Z ONLY. A concurrent agent works ticket NNN
(its files) — never touch those, and never edit <shared files>."

## Done when

- [ ] Each acceptance criterion demonstrated, with the evidence named
- [ ] Fixture protocol followed (prefix, synthetic, cleaned up)
- [ ] Gates green; only owned files modified
- [ ] All work committed on the ticket branch; Codex review approved
```

## Goal file (`planning/goals/NNN-slug.md`)

The delegation packet for one ticket, kept in the repo so a later session can pick the ticket
up. It has three parts, in order:

1. **Wave note.** Parallel-safe with which tickets and why; attended or unattended; dev-server
   port and fixture prefix when the ticket needs them.
2. **Implementer brief.** The filled-in template from `briefs.md`: ticket, goal, done-when,
   worktree and branch, ordered reading list, owned and off-limits files, gates, rules, report
   format. For plan-first tickets it ends with "summarize your plan and stop".
3. **Rulings.** Lead-authored constraints the ticket and spec can't carry: exact class
   treatments, query budgets, fixture prefixes, login etiquette, sign conventions. Added after
   plan review or a fix round, and committed on the feature branch right away.

Why plan-first for large tickets: the plan costs a fraction of the implementation. Reviewing
it catches scope misses (a missing enum member, an invented credential name) before any code
exists, and the Rulings section turns those catches into binding text the go-ahead can point
to.

## Wave plan (`planning/goals/README.md`)

Carries: the team and branch answers from session setup, the worktree caveats (per-worktree installs, env copies, migration serialization, DB contention), the wave diagram, sequencing notes explaining WHY each ordering exists, the attended/unattended split (anything with live logins/OTPs runs attended on the designated machine), and the conventions baked into every brief (gates, Status: review not done, dashboard not regenerated on ticket branches, ticket branches merged into <feature-branch> by the lead and never pushed, one PR from <feature-branch> into <integration-branch> opened by the lead, worktree removal after merge).

```text
Wave A (serial):    001 --> 002          foundation: schema, then auth + shared helpers
Wave B (parallel):  003, 004, 005        after 002 (disjoint owned files)
Wave R0 (serial):   010                  prep ticket, lead-direct; MUST merge first
Wave R1 (parallel): 011, 012 (after 010) each replaces its own stub only
```

## Delivery dashboard (`planning/delivery/dashboard.html`)

A single static HTML snapshot — counts, a progress bar, one table per delivery phase with ticket number/title/status/blocked-by, footer stating "Source of truth: tickets/\*.md — this file is a regenerated snapshot." Regenerate it only on the feature branch, after each ticket merge, so parallel tickets never conflict on it.
