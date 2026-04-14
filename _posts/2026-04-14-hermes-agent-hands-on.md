---
title: "I Actually Installed Hermes Agent. Here's What Happened."
subtitle: "One command, a Telegram bot, and a memory loop that generated its own content brief"
date: 2026-04-14
categories: [Workflow, AI-Agents]
tags: [hermes-agent, ollama, telegram, local-llm, nous-research, hands-on, openclaw]
toc: true
toc_sticky: true
excerpt: "The setup guide I wrote yesterday was based on research. Today I ran the actual installer, connected it to Telegram, and tested whether the memory loop works. The results were more interesting than I expected — including the parts that didn't go as planned."
header:
  image: /assets/images/eyecatch_hermes_agent_handson.png
  teaser: /assets/images/eyecatch_hermes_agent_handson.png
  og_image: /assets/images/eyecatch_hermes_agent_handson.png
---

Yesterday I published [a setup guide for Hermes Agent](https://hybrid-llm.com/workflow/ai-agents/hermes-agent-setup-guide/) based on documentation and user reports. I hadn't actually run it. Today I did.

This is what happened — including the errors, the workarounds, and one test result that genuinely surprised me.

---

## The Install

One command:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Behind the scenes it: installs `uv` (Python package manager), downloads Python 3.11, clones the repo to `~/.hermes`, creates a virtual environment, installs dependencies, downloads Playwright Chromium (165 MB), and syncs 78 skills.

**Total time: around 8 minutes** on my M2 Max. Most of that was the Chromium download.

One thing the setup guide got wrong: the installer ends by launching a setup wizard, which requires interactive terminal input. When you run it via `curl | bash`, stdin isn't a terminal, so the wizard dies with `bash: /dev/tty: Device not configured`. The install itself completes fine — just ignore that error and configure manually.

After install, confirm it worked:

```bash
source ~/.zshrc
hermes --version
# Hermes Agent v0.9.0 (2026.4.13)
```

---

## Connecting to Telegram

The goal: reach Hermes from my phone while the model runs locally.

**Step 1:** Create a bot via [@BotFather](https://t.me/BotFather) — `/newbot`, pick a name, get your token.

**Step 2:** Get your Telegram user ID from [@userinfobot](https://t.me/userinfobot).

**Step 3:** Create `~/.hermes/hermes-agent/.env`:

```
TELEGRAM_BOT_TOKEN=your_token_here
TELEGRAM_ALLOWED_USERS=your_user_id
```

**Step 4:** Start the gateway:

```bash
hermes gateway run
```

Logs confirmed connection in under 15 seconds:

```
✓ telegram connected
Gateway running with 1 platform(s)
```

Send `/sethome` to the bot to register your chat as the delivery target for cron notifications. The bot confirms immediately.

---

## Choosing a Model

This is where the setup guide was optimistic.

I pointed Hermes at `qwen3.5-nothink:latest` via Ollama. First message back:

> Model qwen3.5-nothink:latest has a context window of 32,768 tokens, which is below the minimum 64,000 required by Hermes Agent.

Hermes enforces a 64K minimum. It reads this from Ollama's `/v1/models` endpoint — which reports whatever `num_ctx` is configured for that model. At Ollama's default (32K for qwen3.5), it fails.

There's a workaround: set `num_ctx` to 65536 in your Ollama model config. Hermes will then see 64K and accept it. Whether that's a good idea is a separate question — qwen3.5 is architecturally designed for 32K, so pushing it to 64K via RoPE scaling may degrade quality at longer contexts. Your mileage will vary.

I switched to `gemma4:26b`, which has a native 128K context window and was already running on my machine. No config tweaks needed.

Config in `~/.hermes/config.yaml`:

```yaml
model:
  default: "gemma4:26b"
  provider: "custom"
  base_url: "http://127.0.0.1:11434/v1"
```

After restart, Japanese worked immediately:

> こんにちは → こんにちは。何かお手伝いできることはありますか？

---

## Setting Up the Memory Layer

Hermes has two persistent memory files: `MEMORY.md` (the agent's notes) and `USER.md` (your profile). Both live in `~/.hermes/memories/` and get injected into every session.

I populated them manually to give Hermes the context it needs to act as an editorial layer for HybridLLM-X — what the project is, who reads it, what's being worked on.

I also trimmed `~/.hermes/SOUL.md` down to eight lines. Garry Tan published a post today about AI agent architecture that changed how I thought about this:

> *The harness is the product. The secret isn't the model — it's the thing wrapping the model. Thin harness, fat skills.*

A bloated SOUL.md is a fat harness. The intelligence should live in skill files, not in the system prompt. So the system prompt became minimal:

```
You are the long-term memory and editorial strategy layer for HybridLLM-X.
Your job: receive research, decide what matters, maintain memory, create briefs for Claude Code.
You are NOT the writer. Claude Code writes. You remember and decide.
```

Then I wrote three skill files — reusable markdown procedures — for the actual editorial work:

- **`save-research`** — receives research input, diarizes it, saves to MEMORY.md
- **`create-brief`** — reviews MEMORY.md, picks the strongest angle, outputs a content brief
- **`weekly-review`** — retrospective that updates content strategy

---

## Testing the Memory Loop

This is the part I actually wanted to test: does Hermes work as a long-term editorial brain?

**Test 1: save-research**

I sent Hermes the key ideas from Garry Tan's post — thin harness / fat skills, skill files as parameterized method calls, the 100x productivity claim — and asked it to save the research.

Result: Hermes read the existing MEMORY.md, added a new entry, and wrote it back. It also tried to navigate to X.com to find the original post and verify the source — hit a bot detection wall. Interesting autonomous behavior: it wanted to confirm the source before saving, which isn't in the skill definition. Whether that's a feature or scope creep depends on your perspective.

The entry format wasn't the structured template I defined in the skill file. But the read-then-write loop worked.

**Test 2: create-brief**

```
Create a content brief for the next HybridLLM-X article. format: blog
```

Hermes read MEMORY.md, identified "thin harness / fat skills" as the strongest content angle (correctly — it was the most recent entry with a clear content signal), and produced a full blog brief:

> **Title:** Beyond Orchestration: Why Your Local AI Agents Need "Fat Skills"
> **Goal:** Propose a design paradigm for agentic workflows, moving away from complex orchestration toward intelligent, parameterizable tools.
> **Hook:** The Garry Tan insight. The paradigm shift in agent design.
> **Structure:** Core Thesis → Problem with Heavy Harnesses → Fat Skills Paradigm → Local Advantage → Call to action

The brief was longer than I specified and the format deviated from the skill template. But the editorial logic was sound: it picked the right topic, framed it correctly for the audience, and suggested a structure that would actually work.

---

## What I Learned

**The memory loop is real.** Research goes in via Telegram, Hermes files it, and later retrieves it to make editorial decisions. The loop worked end-to-end on the first try.

**The 64K context requirement is a real constraint — but it's negotiable.** Hermes reads the reported context window from Ollama's API. If your model is configured with `num_ctx` below 64K, Hermes rejects it. You can set `num_ctx: 65536` to pass the check, but models designed for 32K (like qwen3.5) may degrade at longer contexts. Native 64K+ models like `gemma4:26b` are the cleaner path.

**Skill files don't strictly control model behavior — at least not with gemma4:26b.** The output format deviated from the template, and the agent added steps (source verification) that weren't in the skill definition. This isn't necessarily bad, but it means you can't treat skill files as strict contracts. Think of them as behavioral guidelines that the model interprets, not executes literally.

**The bot's autonomous behavior is worth watching.** Hermes trying to verify the Garry Tan source before saving is exactly the kind of initiative that makes agents useful — and exactly the kind of thing that can go wrong at scale. For now it's just interesting. Over time, it'll tell me something about how much to trust the agent's judgment.

---

## Where This Goes Next

The brief Hermes produced is for a separate article about thin harness / fat skills as a design principle for local AI agents. I'll write that one with Claude Code using the brief as the starting point.

That's how the four-part system is supposed to work:
- Perplexity brings the research
- Hermes files it and decides what's worth writing
- Claude Code executes the draft
- OpenClaw routes everything and keeps it running

Today was the first time all four parts were in place. It mostly worked.

---

## Sources

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent)
- [Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/)
- [My setup guide (yesterday)](https://hybrid-llm.com/workflow/ai-agents/hermes-agent-setup-guide/)
- [My user stories piece](https://hybrid-llm.com/workflow/ai-agents/hermes-agent-user-stories/)
