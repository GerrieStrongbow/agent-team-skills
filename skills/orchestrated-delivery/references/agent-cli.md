# Agent mechanics: Codex reviews, headless workers, failure modes

Implementers normally run as subagents through the `Agent` tool (`model: "opus"`,
`effort: "high"`), and fix rounds continue the same agent with `SendMessage`. Codex has no
subagent form, so it runs as a background shell command: the lead starts it, gets re-invoked
when it exits, and reads its output.

Prompts go in on stdin from files in the scratchpad, never inline-quoted, because shell quoting
mangles multi-line prompts. Output goes to files you read afterwards. Model ids are resolved at
session setup (SKILL.md section 0), never copied from a previous project.

## Codex reviewer

```bash
# First review: fresh session, run in the background
codex exec --dangerously-bypass-approvals-and-sandbox -m <sol-model> \
  -c model_reasoning_effort=high -C <worktree> \
  -o <scratch>/review-<ticket>-r1.md - < <scratch>/review-prompt-<ticket>.md > <scratch>/codex-<ticket>-r1.log 2>&1

# Re-review: resume the same session so it remembers its findings
codex exec resume <session_id> --dangerously-bypass-approvals-and-sandbox \
  -m <sol-model> -c model_reasoning_effort=high \
  -o <scratch>/review-<ticket>-r2.md - < <scratch>/rereview-<ticket>.md > <scratch>/codex-<ticket>-r2.log 2>&1
```

`resume` takes no `-C` ("unexpected argument '-C'", exit 2, verified 21 Oct 2026): `cd` into the
worktree first; sessions are filtered by cwd, so run it from the directory the first round used.

The session id is printed near the top of the log. `-o` writes the final message (the verdict
block). The rollout log is the full transcript:
`~/.codex/sessions/YYYY/MM/DD/rollout-*<session_id>.jsonl`. Its timestamps are UTC; convert
them before reasoning about how long something has been running.

Codex can also implement a ticket when the user asks for it: use the same command with an
implementer brief instead of a review prompt.

## Headless Claude worker (when a subagent won't do)

Use a headless session instead of the `Agent` tool when the work must outlive this session,
run attended on another machine, or needs its own transcript file:

```bash
claude -p --dangerously-skip-permissions --model opus --effort high \
  --output-format json < brief.md > out.json 2>&1          # run in the background
jq -r '.session_id, .subtype, .result' out.json            # subtype "success" = clean exit
claude -p --resume <session_id> --dangerously-skip-permissions --model opus --effort high \
  --output-format json < fix-round.md > out-2.json 2>&1
```

Live progress: `~/.claude/projects/<project-dir-slug>/<session_id>.jsonl`.

## Failure modes and recoveries

**Codex credit stall.** With no credits left, `codex exec` hangs silently (no output, almost
no CPU) instead of erroring. The rollout's `token_count` event shows
`"credits":{"has_credits":false}`. Kill it, tell the user in one line, and switch to the fallback
reviewer (SKILL.md section 3) for the rest of the session. Never wait for credits to return.

**Codex finished but won't exit.** The rollout shows `task_complete` but the process idles at
0% CPU and never flushes stdout. Get the final message from the rollout, then kill the PID:

```bash
jq -r 'select(.payload.type=="message" and .payload.role=="assistant") | .payload.content[]?.text' <rollout>.jsonl | tail
```

**Codex session token budget.** If a session stops with "Stopped due to the goal's token
budget limit", `resume` inherits the exhausted budget and refuses. Start a fresh session with
a self-contained brief: what was reviewed, the findings so far, what remains.

**Foreground command cut-offs.** Codex's command host may kill foreground child commands after
about 30 seconds, so long steps (dev servers, batch runs) need explicit handling. If a batch
was cut mid-run, check for half-done state (for example `running` rows) and have it redone.

**Subagent went quiet or ran out of turns.** Its worktree state survives. Send a `SendMessage`
asking for status, or start a fresh implementer on the same worktree with a continuation brief
(what exists, what remains, the constraints restated).

## Operational rules

- One implementer per ticket. Parallel implementers only on tickets with disjoint owned files.
- Work that needs live logins, OTPs or browser profiles runs attended on the designated
  machine. Persistent browser profiles live under the main checkout, since removing a worktree
  deletes its profiles and their device trust.
- Login etiquette is a ruling: one attempt against a real portal; on any rejection or
  challenge, stop and report. The lead decides whether to authorize exactly one clean retry.
- Corrections that change scope go into the goal file's Rulings and are committed before the
  fix-round message.
- Verify completion claims yourself (gates, database state, screenshots). A transcript is
  evidence, not proof.
