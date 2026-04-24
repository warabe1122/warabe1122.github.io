---
title: "A Shared Memory for Hermes and Claude Code"
subtitle: "Hermes's built-in memory, the gap that shows up with a second agent, and a Markdown-only vault that sits outside both"
date: 2026-04-24
categories: [Workflow, AI-Agents]
tags: [hermes-agent, claude-code, memory, vault, local-llm, workflow, agent-framework]
toc: true
toc_sticky: true
excerpt: "Hermes ships with a strong built-in memory system, but it lives inside Hermes. If you drive a second agent (in my case, Claude Code), the memory stays behind. This post describes what Hermes's built-in system does, the gap that shows up in a two-agent setup, and the Markdown-only vault I built outside both agents to close it."
header:
  image: /assets/images/eyecatch_shared_memory_hermes_claude.png
  teaser: /assets/images/eyecatch_shared_memory_hermes_claude.png
  og_image: /assets/images/eyecatch_shared_memory_hermes_claude.png
---

Hermes ships with a working memory system. It auto-injects three Markdown files into every system prompt, curates itself with periodic nudges, indexes past sessions with FTS5, and can create and rewrite its own skills. If you only use Hermes, you don't need anything else.

I use Claude Code as my main driver. Hermes runs in the background — Telegram gateway, cron jobs, voice-memo intake, scheduled summaries. The two agents don't share memory. Every Claude Code session starts with no context from Hermes, and no context from yesterday's Claude Code session either.

This post describes three things: what Hermes's built-in memory does, the gap that appears when a second agent enters the stack, and the Markdown-only vault I built outside both of them. It's not a replacement for Hermes's memory. It's a shared layer that sits on top of Markdown files and happens to be readable by any agent that's told where to look.

## Hermes's built-in memory, in one page

Three files auto-inject into every system prompt:

| File | Contents | Limit |
|---|---|---|
| `SOUL.md` | Persona, boundary rules, role, anti-hallucination rules | Free-form |
| `MEMORY.md` | Environment, workflow conventions, cross-project lessons | ~2,200-character hard cap |
| `USER.md` | User profile, preferences, work style | Free-form |

Five moving parts on top of the files:

- **Periodic nudges.** Hermes proactively asks whether something should be remembered or updated. Curated-with-permission, not dump-everything. Most entries in my `MEMORY.md` arrived through an approved nudge rather than a direct write.
- **FTS5 session search.** Every session is indexed in SQLite. Past sessions are searchable, and cross-session recall is served by LLM-generated summaries rather than raw logs.
- **Autonomous skill creation.** After a sufficiently complex task, Hermes can offer to convert the session into a reusable skill.
- **Self-improving skills.** Existing skills can rewrite themselves as they run. Small fixes accumulate without manual editing.
- **Pluggable external provider.** `honcho`, `mem0`, `hindsight`, `supermemory`, `byterover`, `holographic`, `retaindb`, `openviking`. The `MemoryManager` combines the built-in provider with at most one external provider. A second external provider is rejected to prevent tool-schema conflicts.

Recalled memory is injected inside a `<memory-context>` fence so the model treats it as background data, not user input.

All of this lives at `~/.hermes/`. Other agents don't see it.

## The gap when a second agent enters the stack

Claude Code has its own memory system — user-level `CLAUDE.md`, project-level `CLAUDE.md`, and a per-project memory directory. It is not connected to Hermes.

In practice this means:

- Decisions made in a Hermes session do not reach Claude Code.
- Decisions made in a Claude Code session do not reach Hermes.
- Cross-session state for any one project (hot pricing, active contract version, open questions, recent changes) has to be maintained in two places, or nowhere.

The default failure mode is "nowhere." The human becomes the carrier of truth between agents, retyping yesterday's decisions from memory at the start of each session. For small projects this is fine. For a long-running project with evolving decisions, it produces drift: contract terms that are one version behind, pricing that reflects last week's structure, TODO lists that have already been done.

The fix is to put the hot state somewhere both agents can read.

## Vault layout

The vault is a plain Markdown folder at `~/vault/`. No database, no API, no runtime, no required viewer. Obsidian opens it cleanly — that's what I use day-to-day — but nothing in the vault depends on it. Any editor works. `grep` works.

```
~/vault/
├── Context/
│   ├── MyContext.md      # arrival protocol (index for agents)
│   ├── Profile.md        # stable self, updates monthly
│   ├── Voice.md          # collaboration style
│   ├── Now.md            # this week's focus, updates weekly
│   ├── Setup.md          # hardware, models, tools
│   ├── AI_Handoff.md     # handoff template + in-progress log
│   └── Handoffs/
│       ├── LATEST.md     # most recent handoff snapshot
│       └── <date>.md
├── Projects/
│   └── <project>/
│       ├── STATE.md         # human-maintained, thin
│       ├── INDEX.md         # auto-generated file index
│       ├── SUGGESTIONS.md   # auto-generated digest, appended
│       ├── ROADMAP.md       # optional
│       ├── drafts/
│       │   └── impl_*.md    # human-written impact analysis
│       └── <timestamped notes>.md
├── Resources/
│   └── prompts/
├── Ideas/
└── Inbox/
    ├── _incoming/    # drop zone
    ├── _needs_review/
    ├── _tasks/
    └── _archive/
```

The folder name is deliberately tool-neutral: not `hermes-vault`, not `claude-vault`. In the previous generation the folder was named after the tool that wrote to it. When the tool stopped being the tool, the folder name became a reminder of what used to be. The current name outlives the current tool choice.

## Two reading strategies: arrival protocol and temperature layers

Every agent that enters the vault is told to start at `Context/MyContext.md`. That file is a short index:

- Which files to always read (`Profile.md`, `Voice.md`, `Now.md`).
- Which files to read on demand (`Setup.md` for technical work, `Projects/<name>/` for project-specific work).
- Which folders are deliberately not for reading (`_archive/`).

This is the "arrival protocol." It replaces a whole-vault grep on session start with a deterministic read path.

Context is also separated by temperature — how often the content changes:

| Layer | Example files | Update cadence | Load strategy |
|---|---|---|---|
| Hot persona | `SOUL.md`, `MEMORY.md`, `USER.md`, `CLAUDE.md` | Rare | Auto-inject every turn |
| Stable self | `Profile.md`, `Voice.md` | Monthly | Always read on arrival |
| Current focus | `Now.md` | Weekly | Always read on arrival |
| Technical setup | `Setup.md` | When tools change | Read on technical tasks |
| Project state | `Projects/<name>/STATE.md` | Daily | Read when relevant project is active |
| Project stream | `Projects/<name>/INDEX.md`, notes | Continuous | Read when depth is needed |
| Archive | `Inbox/_archive/`, old drafts | Append-only | Read only if asked |

The rule: the colder the content, the deeper it sits, and the less likely an agent pulls it in without being asked.

## Per-project state triad

Each project folder keeps three files with distinct owners:

- **`STATE.md`** — human-maintained. Deliberately thin. Sections: Focus, In Progress, Stuck, Recent Decisions, Next Up. This is what agents are told to read first for hot project context.
- **`INDEX.md`** — auto-generated list of incoming notes with dates, themes, and one-line summaries. The router writes it.
- **`SUGGESTIONS.md`** — auto-generated, append-only. A local `gemma4:26b` reads new notes against `STATE.md` and writes a short set of proposals.

Ownership discipline is load-bearing. A human writing `SUGGESTIONS.md` ends up writing a plan. An agent writing `STATE.md` ends up restating what it already said in `SUGGESTIONS.md`. Keeping the owners separate keeps each file honest about what it is.

## The automation layer

The pipeline runs on `launchd` every ten minutes:

```
Inbox/_incoming/*        Inbox/*.md (root)      Projects/<name>/
      │                         │                      │
      ▼ normalize.py             ▼ router.py            ▼ synthesize.py
 Voice/YouTube/md    →    classify + route     →   digest (≥3 new notes)
 (yt-dlp + transcript)    (gemma4:26b, T=0.1)    (STATE.md × new → SUGGESTIONS.md + Telegram)
```

Three scripts, one shell wrapper:

- **`normalize.py`** — shapes incoming drops. Voice memos are already transcribed by the time they arrive. YouTube URLs get `yt-dlp` + `youtube-transcript-api`.
- **`router.py`** — classifies each Markdown file with `gemma4:26b` at temperature 0.1 and moves it to the correct `Projects/<name>/`. Rebuilds `INDEX.md` on the way.
- **`synthesize.py`** — triggers when a project has accumulated three or more new notes since the last run. Reads new notes + `STATE.md`, writes a proposal block to `SUGGESTIONS.md`, sends a Telegram notification.

Concurrency is handled with `fcntl.flock`. `launchd` firing during a manual run doesn't double-process.

### The human gate

`SUGGESTIONS.md` is an append-only proposal log. It is not a TODO list. Before implementing any suggestion, I write a short `drafts/impl_<topic>.md` that covers:

1. Which files will change.
2. What could break.
3. How to roll back.
4. Open questions before starting.

Only after that file exists does the work go to Claude Code for implementation.

This is the step where the automation loop stops and judgment begins. `synthesize.py` is allowed to propose. It is not allowed to decide. Hermes's autonomous skill creation is faster than this. It is also harder to undo. The impact-analysis file is a deliberate brake.

## Built-in vs vault: honest comparison

| Axis | Hermes built-in | Vault |
|---|---|---|
| Readable by other agents | No — `~/.hermes/` only | Yes — any agent, any editor |
| Setup cost | Zero — ships working | Meaningful — conventions, router, prompts, launchd |
| Auto-injection | Three files, every turn | Only what the reading agent chooses to read |
| Cross-session search | FTS5 + LLM summarization | Not implemented |
| Autonomous skill creation | Yes, during sessions | No — human gate by design |
| Input surfaces | Conversation + Telegram voice transcription | Markdown, YouTube, external folders (+ voice via Hermes transcription) |
| Tool dependence | Dies with Hermes | Survives any single tool |
| Depth per project | `MEMORY.md` hard-capped at ~2,200 chars; `~/.hermes/memories/projects/<name>.md` uncapped, read on demand | Per-project triad, no cap, read on demand |
| Human-readable | Yes, Hermes-shaped | Yes, tool-neutral |

**Hermes built-in wins on:**
- Speed to value — zero setup.
- Cross-session search — FTS5 across every past session is a real capability the vault doesn't have.
- Autonomy — self-improving skills are a stronger loop than suggest-and-gate.

**The vault wins on:**
- Living outside the agent — other agents can read it.
- Per-project structure — a triad (`STATE` / `INDEX` / `SUGGESTIONS`) with distinct owners, not a single file.
- Non-conversational input surfaces — YouTube URLs and external Markdown folders drop into the same pipe as everything else.
- Tool-swap cost — replace an agent without rebuilding memory.

Single-agent setup: Hermes built-in is enough. Multi-agent setup with a non-Hermes main driver: the trade flips.

## Three agents, one folder

Current usage, after three weeks of running the vault alongside Hermes's built-in memory:

- **Perplexity** — first-pass research in English. Output gets dropped into `Inbox/_incoming/` as Markdown. `normalize.py` and `router.py` handle the rest.
- **Hermes** — editorial judgment, background maintenance, Telegram gateway, cron jobs. Reads `~/.hermes/memories/projects/<name>.md` (its own per-project memory, distinct from `~/vault/Projects/`) before answering questions about a project. The built-in memory system stays active and does what it does. The vault is additive.
- **Claude Code** — writing and implementation. Reads the vault via `CLAUDE.md`. First read is always `MyContext.md`, then whichever project state is relevant to today's task.

No agent runs "on" the vault. They run on themselves. The vault is the piece of shared ground they all happen to look at.

## What's not solved yet

Three gaps remain open:

- **No cross-session search.** Hermes's FTS5 index covers Hermes sessions only. The vault has no equivalent. A full-text index over all Markdown is a small project; I haven't done it yet because `grep` has been good enough.
- **No L2 thematic summarization.** `MEMORY.md` and per-project `STATE.md` can bloat over time. A periodic job that rolls cold entries into `_memory/themes/<theme>.md` is on the list but not built.
- **Suggestion cadence is undertuned.** The threshold of three new notes triggers `synthesize.py` roughly once a day per active project. Some days that's too often. Some days not often enough. Worth instrumenting before adjusting.

## Why this matters for a multi-agent setup

Hermes's built-in memory is strong enough that in a single-agent workflow, you don't need a layer on top. The value of the vault shows up only when a second agent enters the picture and has to participate in the same state.

The smallest version of the vault is useful: an `~/vault/` folder with a `Context/MyContext.md` arrival protocol and per-project `STATE.md` files, read by whichever agent is in the seat today. The automation layer (`inbox-router`, `synthesize`, human gate) is a separate, optional extension on top of that base.

For anyone running a similar two-agent stack — one agent for background automation, one agent for active work — the pattern that's held up is: keep each agent's internal memory intact, and build a shared layer outside both. The shared layer is Markdown. The reading protocol is an index file. The automation is optional. The rest is convention.

---

**Source material**: three weeks of running the vault alongside Hermes's built-in memory and Claude Code's project-level memory. The companion posts on the Openclaw-to-Hermes move are [Hermes Agent Day One: Five Forks in the Road](/workflow/ai-agents/hermes-agent-day-one-forks/) and [Two Weeks of OpenClaw That Never Landed](/workflow/ai-agents/openclaw-2weeks-migration-luggage/).
