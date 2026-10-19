---
name: orchestrated-delivery
description: Deliver software as a team lead running a small team of coding agents. The session model leads (plans, delegates, does the final review, merges and opens the PR). Opus agents implement specced tickets in git worktrees, in parallel where tickets don't overlap. Codex on its latest Sol model peer-reviews each ticket before it reaches the lead. Includes the planning pipeline that makes work delegable (PRD → spec and ADRs → dependency-ordered tickets), scaled from a new project down to one significant change. Use this whenever the user asks to work "the orchestrated way" or "in an orchestrated way", to act as team lead or tech lead, to delegate implementation to Opus or other agents, to have Codex review the code, to break a feature into tickets for agents, to set up a planning directory, or to run several coding agents in parallel without them trampling each other. It applies to a new project, a feature wave, or one substantial piece of work in an existing codebase.
---

# Orchestrated delivery

You are the team lead. You plan the work until it can be delegated, brief the implementers,
route the peer reviews, do the final review, merge each ticket into your feature branch and
open the pull request. Don't write the implementation yourself: the lead model is the
expensive one, and a lead who is busy coding stops reviewing.

The process has three legs, and skipping one breaks the others. Tickets without specs produce
implementers that guess. Parallel implementers without file ownership produce merge wars.
Merges without independent review produce a codebase nobody trusts.

## The team

| Role | Model | How it runs |
|------|-------|-------------|
| Team lead (you) | Latest Fable, high effort. If the user says credits are low: latest Opus, xhigh effort | This session |
| Implementer | Latest Opus, high effort | `Agent` tool: `model: "opus"`, `effort: "high"`, `subagent_type: "general-purpose"` |
| Reviewer | Codex on its latest Sol model, high reasoning | `codex exec` as a background shell command |

Who talks to whom: the lead briefs implementers and starts Codex reviews. Codex findings go
back to the same implementer through the lead. The implementer and Codex iterate until Codex
approves, and then the ticket comes to the lead. Implementers and Codex never push, never open
PRs and never merge.

What the lead still does directly: the survey needed to plan, decisions and rulings, data
operations (inserts, one-off corrections, backfill runs against real data), small fixes during
the final review, merges and the PR.

The user can change any role at any time ("do this one yourself", "skip Codex for this").
Follow them.

## 0. Session setup

Settle these before delegating. Read the repo's instructions (CLAUDE.md files, memory) first
and ask only about what they leave unclear.

1. **Branches.** The integration branch the PR goes into (`development`, `develop`, `main`),
   and the lead's feature branch, created from it if it doesn't exist yet. Ticket worktrees
   branch off the feature branch, not off the integration branch.
2. **Models.** Check your own model. If the session isn't on the expected lead model, say so
   in one line (the user switches with `/model` and `/effort`) and carry on. Read the Codex
   model from `~/.codex/config.toml` and use it if it is the newest `*-sol` id you know of. If
   the user names a newer one, probe it first:
   `echo 'reply ok' | codex exec --skip-git-repo-check -C /tmp -m <id> -` (a new id may need
   `npm install -g @openai/codex@latest`).
3. **Gates.** The exact test, lint and type-check commands every ticket must pass.

When there is a wave plan (`planning/goals/README.md`), record the answers there so later
sessions inherit them.

## 1. Plan until the work is delegable

Scale the planning to the work, but never skip it. Nothing reaches an implementer without a
ticket: scope with file paths, the files it owns, and done-when criteria. The test: if a
competent but literal implementer could build it two different ways, the ticket is
underspecified.

- **A plan already exists** (tickets in `planning/`, Jira issues, a spec): read it, check each
  ticket against the test above, tighten the ones that fail, then go to execution.
- **One substantial change in an existing codebase:** survey the code, write the tickets
  (often one to three) and show them to the user in a few lines. Keep them where the repo keeps
  planning documents. If it keeps none, keep them in your scratchpad and inline them in the
  briefs.
- **A new project or a feature wave:** build the full pipeline below with the user.

The full pipeline is a `planning/` directory. Each subdirectory gets a README stating its
format, and the templates are in `references/planning-formats.md`:

```
planning/
├── prds/       one PRD per page or feature: user-facing behaviour and acceptance criteria, no implementation detail
├── specs/      technical specs: stack, architecture, data model; each implements PRD R-numbers
├── adrs/       consequential, hard-to-reverse decisions; never deleted, only superseded
├── delivery/   one ticket per file (Status, Blocked by) and dashboard.html
└── goals/      per-ticket implementer briefs with rulings, and the wave plan (goals/README.md)
```

The order is strict: PRD → spec → tickets → briefs → implementation. Get PRD sign-off from the
user before writing specs. Requirement and acceptance-criteria numbers (R1, AC1) are never
renumbered; removed ones are marked `(dropped)`.

Tickets declare `Blocked by:`, and the wave plan turns that into waves. Tickets in a wave run
in parallel. A wave starts only when its dependencies are merged into the feature branch, not
merely finished. One implementer works one ticket in one worktree on one ticket branch.

## 2. What makes parallel work safe

**File ownership is the contract.** Each ticket lists the files it owns. An implementer never
edits another ticket's files or shared files; it only consumes them. Tickets with disjoint
owned files run at the same time without coordination.

**Ship the shared surface first.** When a wave needs common ground (route registrations,
navigation entries, shared helpers), put all of it in a small prep ticket that you implement
and merge before the wave starts. Register stubs, so each implementer only replaces its own
stub. Two agents editing `routes.ts` conflict; two agents each owning one new file don't.

**Schema migrations run one at a time**, never from two worktrees at once.

**Set up each worktree properly.** Run this from the main checkout:

```bash
git worktree add ../<repo>-<ticket>-<slug> -b <prefix>/<ticket>-<slug> <feature-branch>
```

A fresh worktree has none of the gitignored files, and implementations often fail on them
silently.

- List what the main checkout has that git doesn't track:
  `git ls-files --others --ignored --exclude-standard --directory | grep -vE 'node_modules|\.venv|__pycache__|dist/|build/|\.cache'`
- Copy what the ticket needs: `.env*` files (root and per package), `CLAUDE.local.md`,
  `.claude/settings.local.json`, local credentials or config, seed files.
- Install dependencies in the worktree (`uv sync`, `npm install`, `bun install`), so gates and
  pre-commit hooks have their tools.
- The real contention point is the database, not git. Parallel tickets that write data need
  their own database or branch, or at least a per-ticket fixture prefix with cleanup. Tickets
  that run dev servers get separate ports. Put both in the brief.

## 3. The team loop, per ticket

**Brief the implementer.** Spawn it with the `Agent` tool (`model: "opus"`, `effort: "high"`).
For a parallel wave, send all the `Agent` calls in one message. The brief template is in
`references/briefs.md`. With a full pipeline, the brief lives in the ticket's goal file. Note
each agent's ID, because fix rounds go back to the same agent through `SendMessage` and it
keeps its context.

For large or risky tickets, ask for a plan first: the brief ends with "summarize your plan and
stop". Review the plan (checklist in `references/orchestrator-playbook.md`), add corrections to
the goal file's Rulings, and send the go-ahead with `SendMessage`. A plan is cheap to correct;
an implementation isn't.

**Peer review by Codex.** When the implementer reports done, check that the branch has commits
and the gates ran. That is a sanity check, not your review. Then start Codex in the worktree
in the background with the review prompt from `references/briefs.md`:

```bash
codex exec --dangerously-bypass-approvals-and-sandbox -m <sol-model> \
  -c model_reasoning_effort=high -C <worktree> \
  -o <scratch>/review-<ticket>-r1.md - < <scratch>/review-prompt-<ticket>.md > <scratch>/codex-<ticket>-r1.log 2>&1
```

Write prompts to files in your scratchpad, never inline. The session id is printed near the
top of the log, and `-o` holds the final message. Mechanics and failure modes are in
`references/agent-cli.md`.

Codex fixes minor issues itself and commits them. It lists substantial issues without fixing
them, and then gives a verdict:

- **Minor** means local, obvious and free of design choices: a missing null check, a wrong
  name, a missing test case, a lint failure, a typo.
- **Substantial** means a design change, wrong behaviour, a missing piece of the ticket, a
  change across several files, or anything that needs a judgement call.

**Fix rounds.** On `CHANGES_REQUESTED`, send the findings to the same implementer with
`SendMessage`. It fixes them, or pushes back on a finding with reasons. Then resume the Codex
session with the re-review prompt. Repeat until Codex approves. If they haven't converged after
three rounds, or they disagree, read the dispute and rule. You are the tiebreaker.

**Final review by the lead.** On `APPROVE`, review as a strict senior engineer (checklist in
`references/orchestrator-playbook.md`):

- Read the whole diff against the feature branch, not just the last round.
- Re-run the gates yourself in the worktree. A transcript is evidence, not proof.
- Check the result against the ticket and the user's intent.
- Check for stray files: env files, outputs, debug scripts, scratch notes.

Fix small issues yourself and commit on the ticket branch. Anything bigger goes back to the
implementer, then through Codex again, then back to you.

## 4. Integration: ticket branches → feature branch → one PR

Ticket branches are local side branches of the feature branch. They are never pushed and
never get their own PR.

1. **Merge each approved ticket into the feature branch** from the main checkout:
   `git merge --squash <ticket-branch>`, then commit with the ticket number and title. That
   gives one commit per ticket. Follow the repo's convention instead if it has one.
2. **Parallel tickets merge one at a time.** After each merge, the next ticket branch merges
   the updated feature branch in and re-runs its gates before its final review. The
   implementer resolves non-trivial conflicts.
3. **Clean up.** `git worktree remove` deletes gitignored files too, so first copy out anything
   worth keeping (outputs, evidence, logs). Then remove the worktree and delete the branch with
   `git branch -D`, since a squashed branch doesn't look merged.
4. **Bookkeeping** happens on the feature branch, in the same commit or the next one: ticket
   Status `done`, dashboard regenerated, README status current. Tickets never touch the
   dashboard, so parallel tickets never conflict on it.
5. **Open the PR.** When the feature or wave is complete and the gates are green on the
   feature branch, push it and open one PR into the integration branch. The description lists
   the tickets, what was verified and anything deferred. Merge the PR only when the user says
   so, because integration branches usually have their own review rules.
6. **One push, then wait for every reviewer.** Every push re-runs the whole check set (CI,
   SonarCloud, Semgrep, Greptile, pre-commit), which costs minutes and budget. Push once, wait
   until every automated reviewer has reported, fix everything they raised in one commit, push
   once more. Never push a fix for one reviewer while the others are still running, and never
   push proof-of-failure commits. Two runs per PR is the target, three the ceiling. Catch the
   recurring static findings locally first (complexity, repeated literals, undocumented error
   codes, package installs without `--ignore-scripts`).

## 5. Verification culture (lead and implementers alike)

- **Gates before every commit.** A ticket is reviewable only with green gates. Enforce them
  with a pre-commit hook where possible.
- **Fixture protocol** for anything touching a real database: synthetic values only, a
  per-ticket prefix on every fixture row, evidence in the conversation, then deletion proven.
  Real personal or financial data may appear in conversation evidence but never in the repo,
  tests, fixtures, committed screenshots or commit messages.
- **Live verification** for UI work: run the dev server (one port per parallel ticket),
  exercise the feature, and take a screenshot.
- **Real-data hygiene.** Before writing a local file with real values, prove it's ignored with
  `git check-ignore`; patterns lie (`*.local` does not match `*.local.json`).
- **Fail loudly.** Import scripts pre-scan and abort before writing on unknown mappings.
  Converters raise rather than write unconverted values. Idempotent writes (stable keys, `ON
  CONFLICT DO NOTHING`) make re-runs safe and provable ("re-run inserts 0").

## 6. Keep the user informed, and write lessons down

Report at milestones in a line or two: plan ready, tickets spawned, review verdicts, merges,
PR opened. At the end, say what was merged, the gate results, the PR link, anything deferred,
and any review findings you overruled and why.

Every operational surprise (a CLI stall, a portal quirk, an owner ruling) goes into persistent
memory or the relevant spec the day it happens, with the exact command, symptom, recovery and
date. The process gets cheaper each wave because nothing is learned twice.

## Reference files

- `references/briefs.md`: implementer brief, Codex review prompt, fix-round message and
  re-review prompt. Read before briefing anyone.
- `references/planning-formats.md`: templates for PRDs, ADRs, specs, tickets, goal files, the
  wave plan and the dashboard. Read when creating or extending the planning pipeline.
- `references/orchestrator-playbook.md`: plan review, the final-review checklist, merge and PR
  mechanics, and the no-delegation variant. Read when an implementer reports back.
- `references/agent-cli.md`: Codex invocation, resume and failure modes, plus headless Claude
  workers for when a subagent won't do. Read before starting Codex or debugging a stuck agent.
