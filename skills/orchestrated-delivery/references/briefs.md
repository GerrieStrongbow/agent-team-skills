# Brief templates

Fill these in and adapt them; delete lines that don't apply. Keep briefs self-contained, since
neither the implementer nor the reviewer has seen your conversation with the user. With a full
planning pipeline, the filled-in implementer brief is the ticket's goal file
(`planning/goals/NNN-slug.md`), followed by its Rulings section.

## Implementer brief (Agent tool, Opus, high effort)

```
You are the implementer on a team. A team lead (me) planned this work. A reviewer will peer-review
your branch when you are done, and I do the final review and merge it into the feature branch.

Ticket: <ticket number and title; path to the ticket file, or the ticket inlined here>
Goal: <what to build or fix, in user terms, and why>

Done when:
- <observable criterion>
- <observable criterion>
- Gates green: <exact commands, e.g. uv run ruff check . && uv run pyright && uv run pytest>

Where:
- Worktree: <absolute path>. Work only there. Branch: <branch>. It is already checked out.
- Env files and dependencies are already in place. If something gitignored is missing, stop
  and tell me; don't recreate secrets.

Read first, in order:
1. <CLAUDE.md / package CLAUDE.md>
2. <the files you will change or imitate>
3. <the ticket, the spec and PRD sections it implements, the goal file's Rulings>

You own: <files or directories>
Don't touch: <files another implementer owns, shared files, migrations unless assigned>

Rules:
- Commit on your branch with clear messages. Don't merge, rebase onto other branches, push or
  open PRs.
- <ports / DB / fixture prefix if running in parallel>
- If a decision isn't covered here and changes behaviour, stop and ask me rather than guess.

Report back with: a summary of what you changed and why, the files changed, the gate output
(last lines), anything you deliberately left out, and open questions.
```

For a plan-first ticket, replace the last paragraph with: "Read the files above, then
summarize the Done-when list and your plan, and stop. I'll review the plan and send the
go-ahead." The go-ahead (`SendMessage`) lists any rulings added since, and repeats the report
paragraph.

## Review prompt (Codex, or the fallback reviewer as an Agent)

```
You are a senior engineer peer-reviewing another engineer's branch before it goes to the team
lead for merge.

Repository worktree: <path>. Branch <branch>, based on <feature-branch>.
Review the change with: git diff <feature-branch>...HEAD and git log <feature-branch>..HEAD.

What the change is meant to do:
<goal and done-when criteria, copied from the implementer brief>

Repo rules to check against: <CLAUDE.md paths, conventions that matter>
Gates: <exact commands>

Review for correctness, edge cases, security, error handling, tests that actually test the
behaviour, consistency with the surrounding code, and anything the goal asks for that is
missing.

Fix MINOR issues yourself (local, obvious, no design choice: null checks, naming, a missing
test case, lint, typos), run the gates, and commit with messages starting "review: ".

Don't fix SUBSTANTIAL issues (design changes, wrong behaviour, missing requirements,
cross-file changes, judgement calls). List them instead.

Finish with exactly this structure:
VERDICT: APPROVE | CHANGES_REQUESTED
FIXED: <one line per minor fix you committed, or "none">
FINDINGS: <numbered substantial issues, each with file:line, what is wrong, and a suggested
fix; or "none">
GATES: <pass/fail per gate>
```

## Fix round (SendMessage to the same implementer)

```
The reviewer looked at your branch and requested changes. It also committed minor fixes (see git log),
so pull them into your view before editing.

Findings:
<paste FINDINGS verbatim>

Address each finding, or push back on one with your reasons if you think it is wrong. Re-run
the gates, commit, and report per finding: fixed (how) or disputed (why).
```

## Re-review (Codex: resume the same session; fallback reviewer: `SendMessage` to the same agent)

```bash
codex exec resume <session_id> --dangerously-bypass-approvals-and-sandbox \
  -m <sol-model> -c model_reasoning_effort=high \
  -o <scratch>/review-<slug>-rN.md - < <scratch>/rereview-<slug>.md > <scratch>/codex-<slug>-rN.log 2>&1
```

```
The implementer has responded to your findings. Their report:
<paste the implementer's per-finding report>

Review the new commits (git log <last-reviewed-sha>..HEAD) and check that each finding is
resolved. Judge disputed findings on their merits. Same rules as before for minor fixes versus
findings, and the same final structure.
```
