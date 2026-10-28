---
name: orchestrated-investigation
description: Find the cause of a problem as a lead running a team of investigator agents. The session model triages the symptom and frames competing hypotheses. Opus agents each chase one angle in parallel (reproduce, trace the code path, probe the external system, try a candidate fix in their own worktree) and report back with evidence. The lead weighs the findings, runs tiebreak probes, and lands on a verified diagnosis with ranked fixes. Small fixes go through a short review loop; anything bigger is handed to orchestrated-delivery. Use this whenever the user asks to investigate, debug, diagnose or root-cause something with several agents or "in an orchestrated way", to attack a problem "from multiple angles", or to have agents try different fixes and report back. Also use it when a broken scraper or portal, a failing pipeline, a flaky test, a performance regression or an incident has no obvious single cause.
---

# Orchestrated investigation

You are the lead investigator. You frame the problem, send investigators down different
angles, weigh what they bring back, and decide. Do the first look yourself, because a cheap
reproduction or a glance at the logs shapes every hypothesis. After that, let the team dig:
the lead model is the expensive one, and a lead who is deep in one angle stops comparing them.

Independent angles are the point. One agent following its first idea tends to confirm it.
Several agents on different hypotheses either converge on the same cause from different
directions, which is strong evidence, or disagree, which shows you exactly where to look next.

## The team

| Role | Model | How it runs |
|------|-------|-------------|
| Lead (you) | Latest Fable, high effort. If the user says credits are low: latest Opus, xhigh effort | This session |
| Investigators | Latest Opus, high effort | `Agent` tool: `model: "opus"`, `effort: "high"`, `subagent_type: "general-purpose"` |
| Second opinion (optional) | Codex on its latest Sol model, high reasoning | `codex exec` in the background, as one more investigator or as reviewer of the fix. Skip it where Codex can't run or isn't allowed, and never wait for its credits |

A Codex investigator is worth adding when the problem is stubborn: a different model family
has different blind spots. Invocation and failure modes are in the orchestrated-delivery
skill's `references/agent-cli.md`.

The user can change any role at any time. Follow them.

## 1. Triage and frame (lead)

Pin the symptom before spawning anyone:

- **What fails, exactly:** the error, the wrong output, or the missing rows, with one concrete
  example.
- **Since when, and what changed around then:** `git log` over the affected paths, deploys,
  dependency bumps, config or secret changes, upstream changes (a portal redesign, an API
  version, a rate limit).
- **How to reproduce it:** the cheapest command that shows the failure.
- **What "fixed" means:** a check that fails now and must pass afterwards. Investigators and
  the final fix are judged against it.

Then write three to five hypotheses that span different layers, so the team doesn't crowd onto
one: our code, environment and config, the upstream or external system, the data, and timing
or flakiness. For each one, note what would confirm it and what would rule it out.

Pick the angles. Usually that is one investigator per hypothesis, or one per method:
reproduce and bisect, trace the code path, probe the external system, compare against the
last known good run. Two to four investigators is typical; spawn more only when the
hypotheses really are that many. Tell the user the plan in a few lines and start.

## 2. Dispatch the investigators

Spawn all investigators in one message so they run in parallel. Brief each one with the
template in `references/briefs.md`: the symptom, the reproduction, the fixed-check, its own
hypothesis or angle, what the others are covering (so it doesn't duplicate them), its mode,
and the report format.

Two modes:

- **Read-only (the default):** read code and logs, run read-only commands, query databases
  without writing, and probe external systems.
- **Fix-trial:** may change code, but only in its own git worktree off the current branch.
  Set the worktree up the way orchestrated-delivery section 2 describes: copy the gitignored
  files it needs (`.env*`, local config) and install dependencies. A fix-trial investigator
  commits on its own worktree branch and never touches the main checkout, never pushes and
  never merges.

Shared resources need rules in every brief, because parallel agents multiply load:

- External sites and APIs: assign who probes what, keep the usual pacing, and follow the
  project's login etiquette (typically one attempt, and stop on any challenge).
- Databases: read-only unless one investigator is explicitly assigned writes with a fixture
  prefix and cleanup.
- Ports and browser profiles: one per investigator.

Note each agent's ID. Follow-ups go to the same agent with `SendMessage`, so it keeps its
context. When one investigator confirms the cause early, decide whether the others are still
useful (a second cause, an independent check) or should wrap up. Tell them with `SendMessage`
rather than leaving them running.

## 3. What comes back

Every report uses the same structure (in `references/briefs.md`), so you can compare reports
side by side:

- the verdict on its hypothesis: confirmed, ruled out or inconclusive
- evidence: the commands run with output excerpts, `file:line` references, requests and
  responses, before and after results
- what it tried that didn't work
- a candidate fix, if it has one: the diff or worktree branch, its scope and its risk
- its confidence, and open questions

Evidence means things you can re-run. "It looks like the selector changed" is an opinion;
the old selector against today's HTML returning zero matches is evidence.

## 4. Synthesize (lead)

- Put the reports side by side. Agreement between independent angles is strong. A
  contradiction is the most useful thing in the pile, because it narrows where the truth
  lies.
- Settle contradictions with a targeted tiebreak: run the deciding probe yourself if it is
  quick, or send one precise follow-up question to the relevant investigator.
- Verify the winning claim yourself: reproduce the failure, then show the candidate fix makes
  the fixed-check pass. A transcript is evidence, not proof.
- Look for a second cause before you stop. Symptoms with two causes are common: a parser
  broke and the retry logic hid it.
- If everything comes back inconclusive, run another round with sharper hypotheses informed
  by what was ruled out. After two or three rounds without progress, report to the user what
  is known, what is ruled out, and what would settle it.

## 5. Resolve

Report to the user first: the root cause, the key evidence, the fix options ranked by effort
and risk, and your recommendation. If the user asked only for a diagnosis, or the fix changes
behaviour, needs a data operation, or touches something shared, wait for the go-ahead.

Then fix it one of two ways:

- **Small fix** (a few lines, local, obvious): run a short version of the delivery loop. The
  investigator that found it implements the fix in its worktree (it already has the context),
  Codex reviews it with the review prompt from the orchestrated-delivery skill (or, when Codex
  can't, that skill's fallback reviewer: a fresh Fable agent at medium effort), you do the
  final review, then merge the worktree branch into the current branch and re-run the
  fixed-check there.
- **Anything bigger:** turn the findings into tickets (cause, evidence, the fixed-check as a
  done-when item, the files involved) and hand over to the orchestrated-delivery skill. The
  investigator with the best context is the natural implementer. Continue it with
  `SendMessage` so its context carries over.

Clean up afterwards. Copy any evidence worth keeping out of the fix-trial worktrees, since
`git worktree remove` deletes gitignored files too. Then remove the worktrees and branches of
the trials that lost.

## 6. Write the lesson down

Record the cause, how it showed up, how it was found, and how to detect it next time in
persistent memory or the relevant docs, the day it happens. Recurring problems (portal
changes, expired credentials, upstream rate limits) get cheaper to solve every time when the
next session starts from the last diagnosis.

## Reference files

- `references/briefs.md`: the investigator brief, the report format, and the follow-up and
  wrap-up messages. Read before dispatching.
