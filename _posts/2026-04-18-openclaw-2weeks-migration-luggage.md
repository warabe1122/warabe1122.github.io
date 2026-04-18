---
title: "Two Weeks of OpenClaw That Never Landed: The Day I Packed for Hermes"
subtitle: "Migration day as an inventory check — and why a thin moving box isn't a defeat"
date: 2026-04-18
categories: [Workflow, AI-Agents, Reflection]
tags: [openclaw, hermes-agent, migration, local-llm, workflow, agent-framework]
toc: true
toc_sticky: true
excerpt: "I packed my box for Hermes and it held almost nothing. Two weeks of OpenClaw, and the things worth carrying over fit in a single cardboard box. I spent the first part of that afternoon treating the thinness as a verdict on me. Then I started reading it differently."
header:
  image: /assets/images/eyecatch_openclaw_migration_luggage.png
  teaser: /assets/images/eyecatch_openclaw_migration_luggage.png
  og_image: /assets/images/eyecatch_openclaw_migration_luggage.png
---

I packed my box for Hermes, and it held almost one cardboard box. Two weeks of OpenClaw, and the things worth carrying over fit in that single box.

I spent the first part of that afternoon treating the thinness as a verdict on me. Then, somewhere between the third and fourth folder I opened and decided I didn't need, I started reading the thinness differently. This post is about that shift, and about the two weeks of OpenClaw that produced it.

It is not a migration-success story. It is not a Hermes-wins story. The market is full of both right now and neither is the thing I actually have to share. What I have is the honest shape of two weeks that never fully became a routine, and the kind of thing moving day tends to expose when you aren't looking for it.

## The day I packed the box

It was the afternoon of Hermes day one. I had spent the morning on the five forks I wrote about in the companion post, and I hit a natural pause around lunch. "While I'm here," I thought, "let me pull whatever's worth carrying over from OpenClaw."

The short list I expected: skill configs, the `projects/` notes I'd been maintaining, a few scripts I'd written that actually fired, maybe a handful of daily logs that had turned into something usable.

The short list I got: less than that.

I'm not going to put a file count on it because the count isn't the point, and counting would make it feel like a scoreboard. What matters is the feel of opening a folder I hadn't touched in ten days and realizing it was a folder I'd created with intent and then never returned to. Most of the folders were like that. A few were worse — ones I'd returned to exactly once, to confirm I wasn't going to come back a second time.

Every one of those folders had been built for a purpose I believed in when I built it. The files weren't sloppy. The ideas weren't wrong. They just hadn't attached to anything the rest of my days actually touched.

That's what "two weeks that never landed" looks like from the inside on moving day: intent-shaped directories with no wear on them.

## What actually happened across those two weeks

I want to be honest about the arc, because the first three days were the ones that made me believe it was going to work.

**Days one to three** were on rails. I installed OpenClaw, wired up my first skill, set up `projects/hybridllm-x.md`, and wrote a couple of scripts that I was genuinely proud of. The setup phase had that good first-few-days-with-a-new-tool feeling where every file you touch feels like it's part of a bigger thing you're building.

**Days four through nine** started to split in a way I didn't notice for a while. Some days I'd open OpenClaw and find a use for it. Other days I'd open my editor, and find myself never needing to open OpenClaw at all. I wasn't avoiding it on those days. I just wasn't reaching for it. The skills I'd written weren't wrong. They were orbiting the edge of a workflow that still started somewhere else.

**Days ten onward** are the ones I'd like to handle carefully. On a few of those days, I noticed something uncomfortable: the day went faster when I didn't open OpenClaw. Not because OpenClaw was slow. Because the friction of crossing the threshold into it — opening Telegram, typing `/restart`, re-briefing the agent on what I was doing — cost more than the value of the trip on those particular days.

I don't know exactly when I started to feel it. There was no single moment. What I know is that by day twelve or thirteen, there was a quiet background feeling that people on X have a pretty good phrase for: "bloated context heavy." I don't think that phrase is exactly right for what I was experiencing, but it's the closest word-shaped object I've seen in the wild, and I recognized it when I read it. A thing that knows too much to meet me where my day starts.

I want to name it and not turn it into a complaint. This wasn't OpenClaw being bad. It was me trying to fit a powerful tool into a workflow that had already been shaped around a different entry point, and quietly losing the fit contest without scoring the game.

## Three reasons it never landed

When I was packing the box on Hermes day one, I could finally see the three things that kept OpenClaw from becoming routine. Two weeks earlier I'd have missed them. On moving day they were obvious — the way packing your things forces you to notice the objects you've been walking around all year.

**The first was startup friction.** On paper, opening Telegram and typing `/restart` takes maybe thirty seconds. In practice, you don't pay that cost once a day. You pay it every time your attention crosses from "I'm doing the work" to "I want the agent to help with the work." That transition happens more times in a day than I expected, and the cost of each crossing compounds in a way that thirty seconds on a stopwatch will never show.

**The second was memory continuity across sessions.** The memory was there. That part worked. What didn't work was the feeling of continuity *between* sessions. Something resumed, but it didn't feel resumed — it felt like reading yesterday's notes that someone else had taken on my behalf. I don't think this is a bug. I think it's an honest reflection of what session-spanning memory looks like when the session boundaries don't match the rhythm of your day. Memory survives the boundary. Continuity, for me, did not.

**The third is the hardest to admit and the most important.** The entry point of my working day is an editor — not a chat surface. That means any tool that wants to become routine has to either live where my editor lives, or pay a toll every time I cross the bridge to get to it. OpenClaw was on the other side of a bridge. A very short bridge, in absolute terms. But every day I chose not to cross.

None of these are criticisms of OpenClaw as a piece of software. They're coordinates. OpenClaw's entry point sat in one place. Mine sat in another. The distance wasn't big. It just didn't close on its own, and I never made it close on purpose.

## What Hermes day one actually showed me

Because my box was thin, I couldn't fill Hermes with two weeks of leftover context. I had to set it up against the shape of the work I was doing *now*, with whatever I'd brought being the whole material.

That was the accident I didn't plan for, and the best thing about moving day.

I wrote a short `SOUL.md` — much shorter than the one I'd accumulated in OpenClaw — because I only had the things I actually carry. I set up one skill first instead of five, because only one of the five had proven itself worth the trip. I pointed it at the projects I was actively working on today, not the ones I'd set up early and forgotten.

This isn't the thing you usually hear about new agent setups. The usual story is that a new tool lets you bring your rich accumulated state and finally put it somewhere worthy. My story was the opposite. The new tool worked because the state I brought was small enough that I had to be honest about what it was.

I want to be careful here, because it would be easy to turn this into a Hermes endorsement, and that's not the argument. "Session-persistent memory" is a good feature, and the token-efficiency claims people throw around (the "roughly a quarter of OpenClaw" number that's been circulating) may or may not hold up in my usage over time. I don't have enough days on Hermes to know. What I do know is that the thing that actually helped on day one wasn't the feature list. It was the fact that I started from a smaller, more honest frame than the one I'd drifted into over two weeks.

That's a property of the packing, not a property of the destination.

## Thin luggage is not a defeat

Here is the reframe I reached by the end of that afternoon, and the whole reason I wanted to write this post.

If you spend two weeks with a tool and pack a thin box on moving day, the obvious interpretation is that you failed to use the tool properly. That's what I believed for about an hour. The reframe is simpler and, I think, more accurate: the two weeks weren't wasted. They were the interval during which I figured out, through actual use, what was worth carrying and what wasn't. The thinness of the box is the *output* of that interval, not the verdict on it.

Without the moving day, that sorting never happens. Without the act of packing, you keep living alongside directories you built with intent and never returned to, and you keep feeling the quiet weight of them without naming it. Moving day is the only thing that makes you open every folder and ask, out loud, whether it's coming along.

Which is why — and this is the thing I want to leave you with — the best advice I can give someone who's currently feeling the same thing about OpenClaw is *not* "migrate to Hermes." It's "pack a box." Pretend you're moving. Open every folder you've built in the last two weeks and decide, one by one, whether it's coming along. You don't have to actually migrate. You just have to do the sorting.

If the box comes out full, great. You have a routine that's grown real weight, and you know what's in it. If the box comes out thin, that's also fine. You've just done the work that two weeks of ambient "I should be using this more" couldn't do on its own.

The thinness of my box wasn't a failure of OpenClaw. It was the first honest inventory I'd taken of my own work in two weeks. I needed the move to do it, because I'm apparently not the kind of person who does it on a regular Tuesday. Maybe you're not either. In that case, the move is the tool, not the destination.

## Honest counter-arguments

**"Isn't this just saying you didn't use OpenClaw properly?"** Pretty much, yes. That's the honest frame. The post is the record of an underuse, not a takedown. If you read it as "OpenClaw doesn't work," you've read it wrong. Read it as "the entry point of my day lived elsewhere, and I didn't close the distance." That's the actual thing.

**"Two weeks is too short to draw any conclusions."** It might be. I've tried to treat this as a moment in a process rather than a verdict. The post says what the first two weeks looked like and what moving day showed me; it doesn't say OpenClaw doesn't have a long-run answer. I'm leaving myself the room to write a revisit piece in two weeks if Hermes turns out to show me the same shape of thing from a slightly different angle.

**"Aren't you just riding the Hermes migration wave everyone else is riding?"** I'd be kidding myself if I said I wasn't aware of the wave. I've read the X threads and the note articles. But the reason I packed the box on day one wasn't the wave — it was the feeling that had been slowly collecting across two weeks. The wave made it easier to notice. It didn't supply the motivation. If it had been any other day on any other tool, I think the packing exercise would have told me the same thing.

**"So should I migrate or not?"** I don't think that's the right question on day two of anyone's reading of this. The right question is whether you've done the packing exercise recently. If you haven't, do that first. The answer to the migration question might resolve itself on its own once the box is in front of you.

---

This post is the third in a short series from verification day, 2026-04-15. The companion posts are [Hermes Agent Day One: Five Forks in the Road](/workflow/ai-agents/hermes-agent-day-one-forks/) and [Don't Let Local LLMs Write Diffs: The L3c Pattern for Fat Skills](/workflow/ai-agents/l3c-pattern-fat-skills/). This one is the after-note — less tactical, more of an honest inventory — and it closes the arc from exhaustion to design to reflection.

**Source material**: two weeks of OpenClaw use (late March through 2026-04-15), one afternoon of packing on Hermes day one, and the companion verification notes from the earlier posts in the series.
