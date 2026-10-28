# Agent team skills

Two [Claude Code](https://claude.com/claude-code) skills for running coding agents the way a software team runs: a lead who plans and reviews, developers who build, and a peer reviewer from a different model family.

I wrote them after typing the same long setup prompt five to ten times in one day. Now one request sets up the whole team.

## The team

| Seat | Model | Job |
|------|-------|-----|
| Tech lead | Claude Fable (your Claude Code session) | Plans, writes the tickets, briefs the developers, routes reviews, does the final review, merges. Writes no code. |
| Developers | Claude Opus, as subagents | Build one ticket each, in their own git worktree. Several run in parallel. |
| Reviewer | OpenAI Codex | Reviews each ticket, fixes small issues itself, and sends anything bigger back to the developer. Work reaches the lead only once Codex approves. When Codex can't run (no credits, not installed, not allowed for the codebase), a fresh Claude Fable agent at medium effort takes the seat, and the lead never waits for Codex to come back. |

The lead not coding is deliberate. It's the most expensive model, so it is spent on judgement, and a lead who is busy coding stops reviewing.

## The skills

**`orchestrated-delivery`** builds things. Planning comes first: a PRD, a spec, then tickets with done-when criteria and a list of the files each ticket owns. File ownership is what lets several developers work in parallel without conflicts. Developers merge into the lead's feature branch, and only the lead opens the pull request.

**`orchestrated-investigation`** finds the cause of a problem. The lead pins down the symptom and writes three to five competing hypotheses, then sends one agent down each in parallel. Each reports back with evidence the lead can re-run. One agent chasing its first idea tends to confirm it; several on different hypotheses either converge or show you exactly where to look.

## Install

Copy the skill folders into your user skills directory (or a project's `.claude/skills/`):

```bash
git clone https://github.com/GerrieStrongbow/agent-team-skills.git
cp -R agent-team-skills/skills/* ~/.claude/skills/
```

Then ask for it in plain words, for example "build this the orchestrated way" or "investigate this in an orchestrated way".

## What you need

- Claude Code, with subagents that accept a model and effort setting.
- The [Codex CLI](https://github.com/openai/codex), logged in, for the reviewer seat. Without it, the lead uses a fresh Fable agent as the reviewer.
- A git repository. Developers work in git worktrees.

If credits are low, the skills fall back to Opus at its highest effort as lead. You can change any seat mid-session ("do this one yourself", "skip Codex for this").

## Before you run it

The reviewer and any headless workers run with permission prompts switched off (`--dangerously-bypass-approvals-and-sandbox` for Codex, `--dangerously-skip-permissions` for headless Claude), because nobody is there to approve them for hours at a time. They work inside their own worktrees, but they can still run any command your user account can. Read the commands in `references/agent-cli.md` before your first run, and use them on repositories and machines where that's acceptable to you.

Model names change often. The skills ask for the latest of each rather than pinning a version.

## Licence

MIT. Use it, change it, share it.
