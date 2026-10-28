# Lead playbook: plan review, final review, merge, PR, bookkeeping

## Reviewing an implementer's plan (plan-first tickets)

The plan is cheap; the implementation isn't. Read the plan against the ticket and spec, looking
for:

- **Scope misses the ticket inherited.** A schema enum missing a member, or a mirror model in
  another component the ticket forgot. Cross-component model mirrors must be in ticket scope
  explicitly; a change on one side that another component mirrors will break the other side's
  validation.
- **Invented facts.** Credential variable names, real personal names, URLs the agent guessed.
  Replace them with role-based names and verified values.
- **Ownership drift.** Any file in the plan that another concurrent ticket owns.

Catches become Rulings in the goal file (or in the go-ahead message when there is no goal
file). Commit goal-file edits on the feature branch right away. If the ticket's worktree
predates the ruling, the go-ahead starts with "merge the feature branch in first".

## Final review (strict senior engineer)

The peer reviewer (Codex or the fallback) has approved by now, but its approval is input, not a verdict. Read the full diff against
the feature branch, not the implementer's summary or Codex's. In rough priority order:

1. **Correctness against the spec's invariants.** Query budgets ("one round-trip, never N"),
   null and blank semantics, sign conventions, division-by-zero paths, idempotency claims.
2. **Independent verification of claims.** Re-run the gates in the worktree yourself. Query the
   database to confirm fixture cleanup (0 rows with the ticket prefix). Check that the links
   and parameters the UI emits match the actual route contracts. Confirm evidence numbers
   reconcile with the source of truth.
3. **Security and data hygiene.** No real values in code, tests, fixtures or commits. No
   secrets. No changes outside the owned files (`git diff --stat <feature-branch>...HEAD`).
4. **Conformance to recorded rulings.** Design-system and architecture rulings beat the
   implementer's taste.
5. **Simplification.** Duplicated helpers get hoisted, or ticketed if cross-cutting.

Fix small defects directly on the ticket branch with a commit message that names the review
reason, then re-run the gates. Bouncing a one-line fix back costs a full round trip; keep
bounce-backs for wrong-approach problems, and send those through peer review again afterwards.

## Merge mechanics (from the main checkout, on the feature branch)

```bash
git switch <feature-branch>
git merge --squash <ticket-branch>
git commit -m "<ticket>: <title>"            # one commit per ticket
cp -R <worktree>/<gitignored outputs worth keeping> <main checkout>/...   # before removal
git worktree remove ../<repo>-<ticket>-<slug> # before deleting the branch; a worktree holds it
git branch -D <ticket-branch>                 # -D because a squashed branch doesn't look merged
```

Merge one ticket at a time when tickets share hot files (registries, config). Resolve
conflicts in the later ticket's worktree (merge the feature branch into it, re-run gates),
never on the feature branch.

## Post-merge bookkeeping (on the feature branch)

1. Ticket `Status: review` → `done`.
2. Regenerate `planning/delivery/dashboard.html` (counts, progress bar, the ticket's row).
   Only ever here, so parallel tickets never conflict on it.
3. Keep the README's status narrative current when code shipped.
4. Commit (`Post-merge NNN: …`). Push only when the PR goes up, or when the user wants the
   branch backed up.

A wave is done when every ticket is merged and bookkept. Only then does the next wave's brief
go out, because waves gate on merged, not finished.

## Opening the PR

When the feature or wave is complete:

1. Run the full gates on the feature branch in the main checkout.
2. Push the feature branch and open one PR into the integration branch. Use the repo's PR
   conventions and template, and the right `gh` account if the machine has several.
3. The description lists each ticket with a line on what it does, what was verified (gates,
   live checks, data checks), anything deferred, and any review finding that was overruled.
4. Don't merge the PR yourself unless the user says so. Integration branches usually have
   review rules and CI of their own. If review comments come back, they become a fix round:
   small ones you fix on the feature branch, bigger ones become a ticket for an implementer.
5. One fix round at a time, and only after every automated reviewer has reported (a `--wait`
   feedback script, or the checks page with nothing pending). Collect everything (CI, Sonar,
   Semgrep, Greptile, people), fix it all, push once. A push made while another reviewer is
   still running throws its result away and re-runs every check. Keep the PR a draft while
   several pushes are unavoidable: a draft runs nothing.

## Direct implementation (the no-delegation variant)

For polish-scale work (styling, small refactors, layout, config), implement directly and keep
the rest: ticket, gates, live verification, self-review, one commit per ticket on the feature
branch. Peer review is optional here; use it when the change touches logic.

Data operations (account inserts, one-off corrections, real-data backfill runs) are always
done by the lead. Scripts go through the normal loop first; then the lead runs them against
the real data with the evidence captured. An implementer never touches real values.
