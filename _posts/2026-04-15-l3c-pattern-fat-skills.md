---
title: "Don't Let Local LLMs Write Diffs: The L3c Pattern for Fat Skills"
subtitle: "Split responsibility across LLM judgment, prompt constraint, and deterministic code — and your skills stop breaking"
date: 2026-04-15
categories: [Workflow, AI-Agents, Patterns]
tags: [local-llm, skill-design, fat-skills, patch-file, gemma4, hermes-agent, agent-framework]
toc: true
toc_sticky: true
excerpt: "The moment I stopped letting gemma4:26b write patch_file calls, my skill stopped breaking. The fix wasn't a bigger model — it was a three-layer responsibility split I'm calling the L3c pattern."
header:
  image: /assets/images/eyecatch_l3c_pattern.png
  teaser: /assets/images/eyecatch_l3c_pattern.png
  og_image: /assets/images/eyecatch_l3c_pattern.png
---

The moment I stopped letting the LLM write `patch_file` calls, my skill stopped breaking.

That's the punchline. Let me give you the correction it needs before it becomes a bumper sticker: this is not a prohibition on `patch_file`. It's a *re-placement* of where the responsibility lives. The model is still in the loop. The model is just not the one writing bytes to disk anymore.

If you're running local LLMs — gemma4, qwen, llama, any of the 7B–30B crowd — and you've written skills that ask the model to edit files, you've probably seen at least one of these three failure modes: a 20-minute streaming hang, a `patch_file` call that produces inconsistent output, or an "edited" file that quietly lost half its existing lines. None of those are bugs in the model. They're the shape of a design error: handing a generative model a deterministic structural job.

This post is about that design error, what a clean split looks like, and a concrete pattern I'm calling **L3c** that you can drop into any skill tonight.

## Why handing diffs to an LLM breaks down

Here's the scenario I actually ran into. I had a fat skill whose job was to update a research log index — append a new entry, remove old ones, maintain date ordering, atomic save. The first version of the skill handed the whole file content to the model inside a long `SKILL.md` prompt and asked it to emit a `patch_file` call with the diff.

It worked the first few times on small inputs. Then it started hanging. Then it started silently deleting old entries. Then — the one that finally got my attention — it produced a "diff" that was actually a fresh rewrite of the whole file with three entries missing and one duplicated.

The problem was not gemma4:26b getting worse. The problem was that the skill was forcing a generative model to be a deterministic structural editor. Generative models are probabilistic by construction. Deterministic edits are not negotiable by construction. You cannot reconcile those two by adding more prompt rules.

## L1 / L2 / L3c — what the three layers actually are

The fix is a split. Every piece of work inside a skill should live in exactly one of three layers:

- **L1 — LLM judgment.** The part where only the model can decide. Example: *"Is this research item stale enough to drop from the index?"* No deterministic rule reaches a clean answer there. The model's strength is exactly this kind of contextual call.

- **L2 — prompt constraint.** The part where you can write a rule inside `SKILL.md` and trust it. Example: *"Date format is `YYYY-MM-DD`. Entries are sorted newest first."* The model still executes this, but the constraint narrows the output space until errors are rare.

- **L3c — code, deterministic.** The part where code can own the work end-to-end with zero model judgment required. Example: physically rewriting a file, atomic rename via `tempfile` plus `os.replace`, deduplication, numbering, log rotation.

A fat skill that tries to do everything through prompts is pushing L3-shaped work through L1. It works until it doesn't. When it stops working, no amount of prompt tuning fixes it, because the problem was never in the prompt.

## A concrete example: `update_index.py`

The skill I refactored is a research log updater. Here's what the new shape looks like.

Instead of `SKILL.md` asking the model to emit a `patch_file` call, `SKILL.md` now describes a *command* the model can shell out to: `update_index.py`. The Python script is about 120 lines. It uses `argparse` for input, `dataclass` for the entry format, and `tempfile` plus `os.replace` for atomic writes.

The contract between the skill and the script is one line of JSON on stdout:

```json
{"status":"ok","count":12,"added":"2026-04-15_new-topic","removed":"2026-04-01_old-topic"}
```

That's it. That's the entire conversation between the script and the model. The model runs the command, reads the JSON, and tells a human "added X, removed Y, 12 entries total." The model never touches the file. The model never writes a diff. The model never emits a `patch_file` call for this work at all.

From the skill's point of view, `update_index.py` is indistinguishable from any other shell command. From the script's point of view, it's just a deterministic CLI that happens to be called by an agent. The two sides meet at a single JSON line.

## What the `SKILL.md` side stopped doing

Here's what the `SKILL.md` looks like after the refactor — not the full file, but the load-bearing part:

- It does not call `patch_file`. Ever.
- It does not call `edit_file`. Ever.
- It does not call `write_file` for this task.
- It carries exactly one explicit rule: *"Do not rewrite the file yourself. Call `update_index.py` instead."*

With `patch_file`, `edit_file`, and `write_file` removed from the skill's available toolbox for this task, the model has nowhere else to route the request. It automatically lands on the L3c option (shell out) because that's the only option left.

Two things happened as a side effect. First, the `SKILL.md` got dramatically shorter — because I no longer had to enumerate every edge case the model might get wrong while patching. Second, the shorter `SKILL.md` made the model's instruction-following accuracy go *up*, not down, which was the opposite of what I expected. I initially worried that pulling context out would make the model lose the plot. What actually happened was that the remaining context was more load-bearing, and the model tracked it better.

This became a virtuous loop. Once one skill looked like this, every other skill I had started wanting to shed its write responsibility too.

## When to use L3c, and when not to

Use L3c for work where code alone can reach the correct answer 100% of the time without a human looking:

- File rewrites
- Log rotation
- State updates
- Deduplication
- Numbering and ordering
- Atomic renames

Do **not** use L3c for work that needs judgment:

- *"Which information should we keep in the index?"*
- *"What's the right phrasing for this summary?"*
- *"What order should these ideas appear in?"*

Those belong to L1. Turning them deterministic is how skills start feeling robotic — and more importantly, it's how you start losing the one thing the model is actually good at.

Rule of thumb: **if a human didn't need to look and code alone would reach the correct answer 100% of the time, it belongs in L3c. Otherwise, it stays in L1.**

## Closing — fat skills aren't the problem. Responsibility placement is.

There's a long-running argument in agent design between "fat skills" (big, self-contained, doing a lot) and "thin harnesses" (small skills chained by orchestration). I'm not going to settle it here, but I'll plant a flag: **fat skills aren't broken. Fat skills with mis-placed write responsibility are broken.** Once you route the deterministic parts to L3c, a fat skill with the rest of its body in L1 + L2 is a perfectly good place to live.

Every test I've run on a 7B–30B local model points the same direction: "fat + L3c-routed" outperforms "thin + LLM writes diffs." The model doesn't have to be smarter. The *structure around it* has to place each job in the right layer.

If you want to try this tonight: pick your single worst skill — the one that hangs, the one that silently loses data, the one you've been meaning to rewrite. Remove exactly one `patch_file` call. Replace it with a Python script that accepts CLI args and emits one line of JSON. Run the skill once. See what changes.

That's the whole experiment. One call, one script, one JSON line.

## Honest counter-arguments

**"If you're writing a script, isn't that just a CLI — why involve the skill at all?"** Correct. The skill's remaining job is "decide whether to call it." That's exactly the point. The skill retreats to being the judgment layer, which is what it's best at.

**"Claude Sonnet handles `patch_file` fine, doesn't it?"** It does. That's why this article is about *local* LLMs specifically. The word "local" in the title is not negotiable. If you're shipping on top of a frontier-class API, the pressure to split responsibilities is much weaker.

**"Isn't `L3c` an overblown name?"** Slightly, yes. But without a name, every "where should this live?" discussion spirals. Naming is an investment in cheaper future arguments. You can use whatever name you want — "L3c" is just the one I'm using.

**"Won't shortening the `SKILL.md` make the model lose context?"** My expectation going in was yes. My observed result was the opposite. Shorter `SKILL.md`, better follow-through. I don't have a clean theoretical explanation, but the empirical answer was unambiguous across a day of testing.

---

**Companion post**: if you want the five-trap onboarding story that surfaced this pattern in the first place, I wrote it up [here](/workflow/ai-agents/hermes-agent-day-one-forks/).

**Source material**: one day of live verification (2026-04-15), the refactored `update_index.py` script (~120 lines of Python), the v2 → v3 `SKILL.md` migration diff, and the research log structure in `projects/hybridllm-x.md`.
