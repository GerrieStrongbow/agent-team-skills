# Brief templates

Fill these in and adapt them; delete lines that don't apply. Keep briefs self-contained, since
investigators haven't seen your conversation with the user.

## Investigator brief (Agent tool, Opus, high effort)

```
You are one of <N> investigators working in parallel on a problem. I'm the lead: I triaged it
and split it into angles, and I'll compare your report with the others'.

Symptom: <what fails, exactly, with one concrete example>
Since: <when it started, and what changed around then, as far as I know>
Reproduce: <the cheapest command that shows the failure, and its current output>
Fixed means: <the check that fails now and must pass after a fix>

Your angle: <the hypothesis to confirm or rule out, or the method to apply>
- Confirmed if: <observation>
- Ruled out if: <observation>
Covered by others, so don't duplicate it: <the other angles, one line each>

Mode: <READ-ONLY | FIX-TRIAL>
- READ-ONLY: read code and logs, run read-only commands, query without writing, probe
  external systems. Don't edit files.
- FIX-TRIAL: work only in <worktree path> on branch <branch> (already checked out, env files
  and dependencies in place). Commit there. Never touch the main checkout, push or merge.

Read first: <CLAUDE.md, the code path involved, recent commits, the relevant logs or outputs>

Shared resources:
- <external site or API>: <who probes what; pacing; one login attempt and stop on any
  challenge>
- Database: <read-only | writes allowed with prefix X, cleaned up afterwards>
- <port, browser profile>

Depth: stop when your hypothesis is confirmed or ruled out with evidence, or after about
<budget> without progress. If you find strong evidence for a cause outside your angle, report
it straight away rather than chasing it.

Report back in exactly this structure:
VERDICT: CONFIRMED | RULED OUT | INCONCLUSIVE
EVIDENCE: the commands you ran with output excerpts, file:line references, requests and
  responses, before and after. Only things I can re-run.
TRIED: what you tried that didn't pan out, one line each
CANDIDATE FIX: none, or a description plus the diff or branch; files touched; risk; whether
  the fixed-check now passes
CONFIDENCE: high | medium | low, and why
OPEN QUESTIONS: anything I should decide or check
```

## Follow-up (SendMessage to the same investigator)

```
Thanks. <Investigator B> reports <the conflicting finding, with its key evidence>. That
contradicts your <claim>. Run <the specific probe that separates the two> and report the
output, with your updated verdict in the same structure.
```

## Wrap-up (SendMessage)

```
<Investigator A> has confirmed the cause: <one line>. Finish what you are doing in the next
few minutes, then report in the usual structure. Include anything that points to a second,
independent cause.
```

## Turning findings into a ticket for orchestrated-delivery

```
# <NNN>: Fix <symptom>

Cause: <root cause, one paragraph>
Evidence: <the two or three decisive observations, re-runnable>
Scope: <files and steps, from the candidate fix>
Owned files: <list>
Done when:
- [ ] <the fixed-check> passes
- [ ] <a regression test that would have caught it>
- [ ] Gates green
```
