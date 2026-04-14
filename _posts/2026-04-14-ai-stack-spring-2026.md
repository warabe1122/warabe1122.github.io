---
title: "My AI Stack in Spring 2026: Four Tools, Four Roles"
subtitle: "A snapshot of how I think about AI right now — before everything changes again"
date: 2026-04-14
categories: [Workflow, AI-Agents]
tags: [ai-workflow, hermes-agent, openclaw, claude-code, perplexity, local-llm, reflection]
toc: true
toc_sticky: true
excerpt: "I've spent the last three weeks wiring local LLMs into my daily work. Somewhere along the way, four distinct roles emerged — and the gaps between them told me more than the tools themselves."
header:
  image: /assets/images/eyecatch_ai_stack_2026.png
  teaser: /assets/images/eyecatch_ai_stack_2026.png
  og_image: /assets/images/eyecatch_ai_stack_2026.png
---

This isn't a comparison article or a setup guide. It's a thinking-out-loud memo — a record of how I understand AI tools right now, in spring 2026.

I'm writing it down because I know this picture will change. A year from now, the tools will be different, my workflow will have shifted, and some of what I believe today will look naive. That's exactly why it's worth capturing.

---

## Where This Started

I've been running Claude Code as my primary work environment for a while. Projects, writing, code, scheduling — everything flows through chat. It's comfortable. The act of typing what I want and watching it happen never gets old.

But after a few weeks of serious use, a pattern started bothering me.

Every new session starts from scratch. I'd restate context. Re-explain project goals. Copy-paste things I'd already said two days ago. When I had multiple projects running in parallel, the boundaries blurred — which conversation had which assumption? Which decision was made where?

Claude Code is great at doing. It's less great at remembering why.

That friction is what led me to look seriously at Hermes Agent. Not for the features list, but for one specific design choice: persistent memory.

---

## What "Persistent Memory" Actually Means

When I first heard "persistent memory," I imagined a giant log file. Everything you've ever said, stored and searchable. Basically a vector database with a chat interface.

Hermes does something different. Its memory works in layers:

- **Working memory** — the current conversation, like any chat session
- **Long-term memory** — structured files like `MEMORY.md` and `USER.md` that persist across sessions
- **Search layer** — SQLite with full-text search, where past conversations can be retrieved and summarized

The key insight: Hermes doesn't just store your conversations. It *distills* them. Past interactions get searched, summarized, and written back into the long-term memory files. The agent doesn't remember everything you said — it remembers what mattered, in an organized form.

This is closer to how a good executive assistant works. They don't transcribe every meeting. They update the briefing doc.

Once I understood that, something clicked: this kind of memory isn't about chat history. It's about **strategy**.

---

## Persistent Memory Is a Strategy Tool

The things that benefit most from persistent memory aren't daily tasks. They're the slow-moving, high-context decisions:

- What's my content strategy this month?
- Which themes am I pursuing? Which ones did I drop, and why?
- How should X, blog, YouTube, and newsletter divide their roles?
- What worked last week? What didn't land?

These are exactly the things that get lost between sessions in a tool like Claude Code. I can write them into `MEMORY.md` manually — and I do — but maintaining that by hand is work. Hermes automates the distillation.

That's when the role split became clear in my head:

> Long-term strategy and periodic review → Hermes.
> Short-term execution and output → Claude Code.

Not competing tools. Different layers of the same workflow.

---

## The Four Roles

Once I started thinking in layers, four distinct roles emerged. Each one maps to a tool I'm already using or plan to use.

### Perplexity: The External Researcher

Web, news, papers, social threads — Perplexity scans the outside world and brings back structured results with sources attached. It's the tool I reach for when I need to know "what's happening out there."

It doesn't hold context. It doesn't remember last week's query. It's a pure research function — fast, broad, disposable.

### Hermes Agent: The Long-Term Strategist

Hermes takes what Perplexity finds and decides what matters *to me*. It writes the important parts into long-term memory files. Over time, it builds a picture of my projects, my priorities, my editorial direction.

During weekly or monthly reviews, Hermes is the one that says: "Here's what you've been working toward. Here's what shifted. Here's what to focus on next."

It's not an executor. It's an editor-in-chief.

### Claude Code: The Craftsman

Claude Code gets a brief — purpose, audience, key points — and turns it into actual output. Article drafts, code, scripts, thread copy, multi-format expansions. It's the one that does the work, in the concrete sense.

This is where most of my day happens. But the quality of what Claude Code produces depends heavily on the quality of the brief it receives. That's Hermes's job.

### OpenClaw: The Hub

OpenClaw connects everything. It receives inputs from Telegram, X, email, cron triggers. It routes tasks to the right destination:

- Research request → Perplexity
- Long-term note → Hermes
- Immediate task → Claude Code

It also handles scheduling: weekly Perplexity research runs, monthly Hermes reviews, daily task automation. It's the orchestration layer — not smart on its own, but essential for keeping the pieces connected.

---

## The Content Workflow

When I map this to my actual content production process, it looks like this:

```
[1] Input — an idea, a link, a thread, a thought
     ↓
[2] OpenClaw — receives it, routes it
     • Research needed → Perplexity
     • Long-term note → Hermes
     • Quick task → Claude Code
     ↓
[3] Perplexity — returns structured research with sources
     ↓
[4] Hermes — filters what matters, updates long-term memory
     • "This theme is worth pursuing"
     • "This connects to the project from last month"
     ↓
[5] Hermes — during periodic review, decides what to create next
     • Produces a brief: purpose, audience, key arguments
     ↓
[6] OpenClaw — sends the brief to Claude Code
     ↓
[7] Claude Code — writes the draft, iterates, expands to formats
     • Blog post, X thread, newsletter, script
     ↓
[8] Publish — feedback loops back through OpenClaw
     • Hermes records what resonated and what didn't
     → feeds into the next strategy cycle
```

This isn't theoretical. Steps 1-7 are roughly what happened with the Hermes articles I published this week. The missing piece is step 8 — closing the feedback loop automatically. Right now, I do that manually.

---

## What Humans Still Own

Laying out these roles made something else clearer: what's left for me.

**Vision and values.** No tool decides what I care about or where I want to go. That's the starting condition everything else depends on.

**System design.** Which tool gets which role? Where does a human need to intervene? When should automation stop and judgment begin? These are design decisions, not execution tasks.

**Final call.** When Hermes suggests a direction or Claude Code produces a draft, someone has to say "yes, this" or "no, try again." That someone is me. AI proposes. I dispose.

Thinking about what to delegate is also thinking about what I refuse to let go of. That turned out to be a surprisingly useful exercise.

---

## The Gaps That Still Exist

Here's the honest part. This four-role framework looks clean on paper, but the connections between tools are still mostly manual.

- When Perplexity finds something relevant, *I* decide to save it to Hermes.
- When Hermes surfaces a strategic insight, *I* write the brief for Claude Code.
- When OpenClaw receives a message, *I* decide where it should go.

The tools don't talk to each other natively. There's no shared protocol, no automatic handoff. The human is still the integration layer.

I don't think this lasts long. Common protocols, persistent memory standards, agent-to-agent coordination — these are all moving fast. But right now, in April 2026, the gaps are real and they cost time.

Knowing that is useful too. It tells me exactly where to watch for the next breakthrough.

---

## A Snapshot, Not a Conclusion

This memo isn't advice. It's a timestamp.

Three weeks of running local LLMs. A few days of digging into Hermes. Two articles published. One setup completed. And now this — a record of how the pieces fit together in my head, right now.

Some of this will hold up. Some won't. The workflow diagram might look completely different by summer. A new tool might collapse two of these roles into one. That's fine.

The point of writing it down isn't to be right. It's to have something to compare against later — to see how the thinking evolved, what assumptions I was carrying, and which ones I eventually dropped.

If you're building your own AI workflow, maybe this gives you a framework to start from. Or maybe it gives you something to disagree with. Either way, I'd rather show the thinking in progress than wait until it's "finished."

It's never finished.
