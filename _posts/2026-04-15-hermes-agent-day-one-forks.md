---
title: "Hermes Agent Day One: Five Forks in the Road, Not Five Bugs"
subtitle: "A full day of running Hermes Agent for the first time, framed as the choices you will also have to make"
date: 2026-04-15
categories: [Workflow, AI-Agents]
tags: [hermes-agent, openclaw, local-llm, gemma4, onboarding, agent-framework]
toc: true
toc_sticky: true
excerpt: "I ran Hermes Agent for a full day for the first time. The five places I tripped aren't bugs — they're forks in the road every adopter walks through on day one. Here's each fork, and which side I picked."
header:
  image: /assets/images/eyecatch_hermes_day_one_forks.png
  teaser: /assets/images/eyecatch_hermes_day_one_forks.png
  og_image: /assets/images/eyecatch_hermes_day_one_forks.png
---

At 8 AM I typed `/restart` into Telegram, and by noon I had moved almost nothing forward. Every single reason was already in the docs.

That sentence is not an exaggeration. I spent a full day running Hermes Agent myself for the first time, and five separate things tripped me. Four of them were explicitly documented. All of them looked, in the moment, like "this thing is broken."

None of them were.

Halfway through the day I noticed a pattern that changed how I read my own failures. The five places I got stuck were not accidents. They were *forks in the road* — decisions that anyone adopting Hermes will be forced to make on day one, whether or not they realize they're making them. Each fork has two plausible sides. Neither has a clear default. If you're setting Hermes up tonight, you are going to walk through the same five choices, and my answers may or may not be yours. The point isn't what I picked — it's noticing the picks exist.

Here are the five.

## Fork 1 — `/restart` vs. `/new`

**What happened**: `/restart` restarts the gateway daemon while preserving your conversation. `/new` opens a fresh session. I spent half a day ambiently uneasy that "Hermes isn't forgetting yesterday" before I figured out that `/restart` was doing exactly what it's supposed to do and I was the one using it wrong.

**The fork**: do you start every session with `/new` for a clean slate, or stay on `/restart` and carry state across so you move faster? Both have real upside. A fresh session is contamination-free but forces you to re-brief the agent on whatever you were working on. A preserved session keeps the agent's working context intact and lets you pick up where you stopped, but it also means yesterday's half-wrong assumption is still sitting in the window, quietly shaping today's reasoning. There is no default here. You are forced to pick a side.

**What I picked**: `/new` exclusively during verification days. `/restart` during normal operation. The genuinely difficult zone is when "I'm testing" and "I'm running the thing" blur into each other — which is most days.

**Do this**: if you're testing, open with `/new` every time. Earn your way back to `/restart` only when you're firmly in operation mode.

## Fork 2 — Reuse old session files, or start clean every time

**What happened**: I carried attachments and context from earlier sessions into a new one, figuring it would save setup time. What it actually did was let yesterday's stack trace silently shape today's judgment. At one point I was debugging a problem that had been fixed hours earlier, because an old error message was still quoted in context.

**The fork**: do you keep session logs so the agent can learn from past failures, or wipe them every time to cut contamination? There's a hidden assumption most adopters start with — that "training material" and "contamination source" are different files. They aren't. They're the same file. You cannot fully have both.

**What I picked**: clean. Past failures get summarized by me (the human) and rewritten into `projects/*.md`. That's slower than auto-carry, but it means every piece of past signal that reaches the agent has been filtered through a human intention.

**Do this**: fresh verification goes into a fresh session. Past signal reaches the agent through *human-written* summaries, not through auto-carried context.

## Fork 3 — Back up old skills, or delete them outright

**What happened**: I had two directories sitting next to each other in my skills folder — `hybridllm-x/` (the live version) and `hybridllm-x.pre-verify/` (a backup I'd made before a refactor). Both were getting loaded. The stale `SKILL.md` was winning the resolution race, and a bug I had "already fixed" came back without warning. It took me longer than I'd like to admit to notice.

**The fork**: when you update a skill, do you keep the old version as a backup, or delete it? Keep it → you must move it somewhere the skill loader will not see. Delete it → you accept that you lose your undo rope. Both are defensible. The trap isn't "backups are bad." The trap is leaving the backup *inside the load path*.

**What I picked**: move backups out of the load path entirely — example target, `~/.hermes/_archive/`. But here's the recursive catch: this only works if you've first read your loader's path resolution rules. Otherwise "outside the load path" is guesswork.

**Do this**: before you run any verification, confirm the skill load path once. Backups belong in a directory the loader cannot see.

## Fork 4 — Reload bootstraps by hand, or trust the automation

**What happened**: OpenClaw ships a bootstrap watcher (a LaunchAgent) that already watches `SOUL.md`, `USER.md`, and `MEMORY.md` and reloads them automatically whenever you edit them. I did not know this. I `launchctl`'d the bootstrap by hand after every edit because it felt safer, triggered a double reload, and briefly killed the gateway.

**The fork**: do you hold bootstrap reloads in your own hands, or delegate to the watcher and stay in observe mode? This is a textbook "manual comfort vs. trust the automation" fork. The honest answer depends on whether you've read enough of the watcher's behavior to trust it — and on day one, nobody has.

**What I picked**: automation owns it. I still watch the log output (in my case, `/tmp/openclaw_bootstrap_reload.log`) every time I edit a bootstrap file. Observe, do not touch.

**Do this**: when reading the log is enough, don't reach for the lever. Reaching for the lever is what caused the outage, not the automation.

## Fork 5 — gemma4:26b token collapse on long context

**What happened**: with a `SOUL.md` around 13 KB stacked on top of a long user message, gemma4:26b hit token degeneracy mid-stream and the response just… hung. I waited 20 minutes before I noticed. The model was still technically "generating" — it was emitting garbage tokens at a pace that would never terminate in any reasonable window.

**The fork — and this one has three sides**:

- **(a)** move up to a bigger model
- **(b)** shrink `SOUL.md`
- **(c)** pull write responsibility out of the skill entirely so the prompt gets shorter and the model's job gets narrower

All three work. They cost different amounts of money and they teach you different things about your setup. (a) is the reflex answer for most people; (b) is the "prune what you have" answer; (c) is the structural one.

**What I picked**: (c). Pulling writes out of the skill made the skill shorter, made the model's job narrower, and — almost as a side effect — became the seed of the design pattern I'll write up next. The streaming hang stopped. The skill stopped breaking. Nothing about gemma4 itself changed.

**Do this**: before you upgrade the model, check whether a single long-context skill is doing too many jobs. Separate responsibilities first. Change hardware second.

## Closing

These five ruts are not a record of accidents. They are a record of choices — choices I was making without noticing, like most people setting up any new tool for the first time. My answers aren't guaranteed right. But if you're reaching for Hermes tonight and you recognize "oh, I'm making one of those right now," this post has already saved you ten minutes.

In the companion post, I take choice (c) from fork 5 and turn it into a structural pattern I'm calling **L3c** — a way of splitting the responsibilities inside a fat skill so the local LLM never writes diffs. If the streaming-hang section above resonated with you, that's the article to read next.

## Honest counter-arguments

**"Isn't this just RTFM?"** Yes. But if RTFM prevented everything, none of this would have happened in a single day. "It's in the docs" does not equal "no reader will hit it."

**"Aren't these all setup-specific to you?"** Four of them are structural and will hit anyone running Hermes and OpenClaw together. Only the gemma4 one is model-specific — and even there, the underlying fork ("where does write responsibility live?") generalizes to any local LLM.

**"So are you recommending Hermes or not?"** Yes. Strongly. But budget four hours for day one. That's the honest answer.

---

**Source material**: one day of live verification (2026-04-15), `projects/hybridllm-x.md`, the bootstrap reload log, and Telegram traces from the verification phases.
