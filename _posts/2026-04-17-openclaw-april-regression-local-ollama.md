---
title: "Is OpenClaw Actually Broken This April? Two Weeks of My Local Ollama Logs"
subtitle: "A first-person post-mortem of 131 LLM timeouts, a 40-minute Ollama request, and a 60-second idle timeout that hit local users the hardest"
date: 2026-04-17
categories: [LocalLLM, OpenClaw, Diagnostics]
tags: [openclaw, ollama, local-llm, timeout, regression, debugging, m2-max, post-mortem]
toc: true
toc_sticky: true
excerpt: "OpenClaw has been brittle since late March. On my local Ollama setup I can now point at the exact line of code, the exact minute it went wrong, and the three-knob fix — scoped strictly to what my own logs prove."
header:
  image: /assets/images/eyecatch_openclaw_april_regression.png
  teaser: /assets/images/eyecatch_openclaw_april_regression.png
  og_image: /assets/images/eyecatch_openclaw_april_regression.png
---

Something is clearly wrong with OpenClaw in April 2026.

A high-profile user described it on X as effectively unusable over the past week. The timeline has been filling up with variants of the same complaint. My own gateway log, over the same window, has 131 `FailoverError: LLM request timed out` lines in it.

This is not a defense of OpenClaw and it is not an attack on it. It is a post-mortem of **one specific failure mode**, scoped to what I can see on my own machine: a Mac Studio M2 Max, 64 GB RAM, running a local Ollama stack that OpenClaw fans out to across four agent roles (chat, orchestrator, writer, research).

Other people's symptoms — 30-minute UI hangs, database save failures, Google auth disconnects, problems against cloud providers — I am not going to claim to explain. I explicitly can't. Those are different failure modes, and I will draw that line hard in the last section.

What I *can* do is tell you, with line numbers and timestamps, what broke inside the local-Ollama path, when it broke, and how I'm reading the evidence.

---

## What "broken" looked like on my box

I run OpenClaw as a daemon. It has been up continuously since late March.

Between **2026-04-03** and **2026-04-17** — roughly the period when the community started sounding alarmed — the gateway stderr log recorded 131 events of exactly one shape:

```
2026-04-03T18:31:54.253+09:00 [diagnostic] lane task error:
  lane=main durationMs=62762
  error="FailoverError: LLM request timed out."
```

The distribution of those 131 events by day:

| Date | Events |
|------|--------|
| 2026-04-03 | 59 |
| 2026-04-04 | 14 |
| 2026-04-05 | 12 |
| 2026-04-06 | 3 |
| 2026-04-07 | 3 |
| 2026-04-08 | 18 |
| 2026-04-10 | 2 |
| 2026-04-15 | 10 |
| 2026-04-17 | 10 |

Two things jump out from this table.

First, the `durationMs` clusters tightly around **62,000 ms ± 3 s**. Not 30 s, not 90 s — almost exactly 60 s, with a few seconds of setup overhead. Whatever is cutting these requests off, it is cutting them off on a fixed 60-second ceiling.

Second, the distribution is not uniform. It is lumpy in a way that strongly implied, before I dug further, that the failures were not "the model is slow" but "something is getting queued behind something else." Clean days are clean. Bad days are very bad.

That second observation is the one I want to follow.

---

## The smoking gun: one minute of Ollama log that explains everything

OpenClaw's stderr tells me it timed out. Ollama's server log tells me *why*.

Here are four consecutive entries from `~/.ollama/logs/server.log`, all resolving within one second of each other on the night of 2026-04-15:

```text
[GIN] 2026/04/15 - 00:52:00 | 200 |   41m46s | POST "/v1/chat/completions"
[GIN] 2026/04/15 - 00:52:00 | 500 |   22m23s | POST "/v1/chat/completions"
[GIN] 2026/04/15 - 00:52:00 | 500 |   19m20s | POST "/v1/chat/completions"
[GIN] 2026/04/15 - 00:52:00 | 500 |   19m21s | POST "/v1/chat/completions"
[GIN] 2026/04/15 - 00:52:01 | 499 |   38m12s | POST "/api/chat"
```

Read the durations. Five requests that started at least 22 minutes apart — the 41m46s one began at roughly 00:10, the 38m12s one at 00:13, the 22m23s one at 00:29, the two 19-minute ones at 00:32 — all terminating within the same one-second window at 00:52. The only way that happens is if later requests sat blocked until earlier ones cleared.

That is not five slow models. That is **one model answering five serialized callers**, each caller blocked until the previous one released the slot.

This is what `OLLAMA_NUM_PARALLEL=1` looks like from the outside.

Ollama's defaults on my machine:

```
OLLAMA_NUM_PARALLEL:       1
OLLAMA_CONTEXT_LENGTH:     65536
OLLAMA_KEEP_ALIVE:         30m0s
OLLAMA_MAX_LOADED_MODELS:  3
```

OpenClaw, meanwhile, will happily fan out to four agents in parallel on a single turn — chat and orchestrator both want their thinking model, writer wants its prose model, research wants its summarizer. When those four lanes all hit Ollama at once and Ollama can only run one at a time, three of them wait. On a long turn, they wait a *long* time.

And here is where the 60-second ceiling from the previous section starts to matter.

---

## The 60-second idle timeout: what changed, when, why

OpenClaw has always streamed tokens from its providers. Until **v2026.3.31**, if the model was slow but eventually producing output, the stream would finish.

In v2026.3.31 a refactor changed that. The current source, readable in the installed package, looks like this:

```js
// ~/.npm-global/lib/node_modules/openclaw/dist/pi-embedded-bukGSgEe.js:34598+

// Default idle timeout for LLM streaming responses in milliseconds.
// If no token is received within this time, the request is aborted.
// Set to 0 to disable (never timeout).
// Default: 60 seconds.
const DEFAULT_LLM_IDLE_TIMEOUT_MS = 6e4;

function resolveLlmIdleTimeoutMs(cfg) {
  const raw = cfg?.agents?.defaults?.llm?.idleTimeoutSeconds;
  if (raw === 0) return 0;
  if (typeof raw === "number" && Number.isFinite(raw) && raw > 0)
    return Math.min(Math.floor(raw) * 1e3, MAX_SAFE_TIMEOUT_MS$1);
  return DEFAULT_LLM_IDLE_TIMEOUT_MS;
}
```

Two things matter here.

**This is an *idle* timeout, not a total timeout.** The request aborts if no token is received for 60 seconds. Fast requests aren't penalized. Long requests that are steadily streaming aren't penalized. What *is* penalized is **requests that sit silently in Ollama's queue for 60 s before their first token**. Which is exactly what happens when three other agents are ahead of you in a `NUM_PARALLEL=1` queue running a 65K-context prefill.

**It is configurable, but discoverability is weak.** The parameter *is* documented — `docs/concepts/agent-loop.md` in the main repo describes `agents.defaults.llm.idleTimeoutSeconds`. But a user hitting the symptom ("my agent stops talking after about a minute") is unlikely to navigate there on the first try. I found it by grepping the installed dist after my logs told me the ceiling was 60 s; that is the wrong order of operations for a default that silently aborts heavy local-model setups.

Note also a docs/runtime drift worth flagging: the current docs state that when `idleTimeoutSeconds` is unset, OpenClaw falls back to `agents.defaults.timeoutSeconds`, or 120 s otherwise. In the v2026.4.1 dist on my machine, `resolveLlmIdleTimeoutMs()` returns the hardcoded `DEFAULT_LLM_IDLE_TIMEOUT_MS` (60 s) in the unset case — no fallback to the outer timeout. Whether the docs describe a newer runtime behavior or an intended-but-not-shipped one, I can't tell without reading the head branch. What I can tell is that relying on "docs-as-truth" without reading your installed `dist` will burn you.

The community bisect thread ([issue #63200](https://github.com/openclaw/openclaw/issues/63200)) points at [PR #55072](https://github.com/openclaw/openclaw/pull/55072) (`feat: add LLM idle timeout for streaming responses`, merged 2026-03-30 by @liuy) as the PR that landed the 60 s default. The last version widely reported as stable is **v2026.3.28**. v2026.3.31 through v2026.4.13 carry the default in its original form.

v2026.4.14 ships [PR #66418](https://github.com/openclaw/openclaw/pull/66418) (`fix(agents): honor embedded ollama timeouts`, by @vincentkoc), which closes [issue #63175](https://github.com/openclaw/openclaw/issues/63175). Read the PR description carefully: it fixes a *different* layer — embedded runs were initializing the global undici stream timeout at a hardcoded 30-minute default instead of honoring the operator-configured run timeout. It does not change the 60 s idle-timeout default. Community reports in [issue #67334](https://github.com/openclaw/openclaw/issues/67334) document operators on v2026.4.14 who set `idleTimeoutSeconds: 600` and still see timeouts, which is consistent with the v2026.4.14 fix targeting a separate seam from the one I'm about to describe.

I have not personally run a full version bisect on my own machine; the version claim above leans on the community bisect, not on my own installations. What I *can* show on my own box is the current state: I'm on **v2026.4.1**, which sits squarely in the affected range, and the timeout signature I have is exactly what that regression produces.

---

## The three-knob fix, in the order I would apply it

Here is the ranked remediation. In the order I would actually do it, based on my logs.

### Knob 1: Raise `idleTimeoutSeconds` in `~/.openclaw/openclaw.json`

```json
{
  "agents": {
    "defaults": {
      "llm": {
        "idleTimeoutSeconds": 600
      }
    }
  }
}
```

600 s is ten minutes. This does not "fix" slow requests. It stops OpenClaw from *aborting* requests that are merely queued. On a local Ollama setup with heavy models, ten minutes of queue wait is not pathological; it is Tuesday.

This is the knob I expect to have the biggest immediate effect on the `durationMs≈62000` failure signature documented above, because it addresses the mechanism directly: the 60 s abort is what creates the timeout events, and raising the abort threshold deprives the failure of its trigger. Knobs 2 and 3 below address the *cause of the wait* rather than the abort itself; they are complementary, not substitutes.

### Knob 2: Raise `OLLAMA_NUM_PARALLEL` to 3

```bash
# ~/.ollama/config.env or launchd plist
OLLAMA_NUM_PARALLEL=3
```

This is the root cause fix. With three parallel slots, the common case — chat + orchestrator + writer all asking for a token at the same time — stops serializing. Writer no longer waits 20 minutes for chat.

On an M2 Max with 64 GB RAM and three `qwen3.5-nothink` instances that want roughly 8 GB apiece, three is the ceiling I'm comfortable with. You may be able to push higher; measure residency against `OLLAMA_MAX_LOADED_MODELS`.

### Knob 3: Set `UV_THREADPOOL_SIZE=64` for OpenClaw's Node process

Node's default is 4. When OpenClaw streams through four agent lanes simultaneously, the libuv thread pool becomes the bottleneck even when Ollama has capacity. The symptom looks identical to a timeout: lanes that *should* be streaming sit silent.

In the launchd plist that runs OpenClaw:

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>UV_THREADPOOL_SIZE</key>
  <string>64</string>
</dict>
```

I would apply this last. Knobs 1 and 2 target the failure signature in my logs directly; knob 3 is insurance against a secondary bottleneck that only tends to bite on heavy multi-agent chains. For a lighter workload, you may never need it.

**Apply in the order 1 → 2 → 3.** Knob 1 stops the bleeding. Knob 2 removes the cause. Knob 3 is defence in depth.

---

## What this post-mortem does not explain

I want to be very explicit about what this diagnosis does not cover, because the public conversation is conflating several different problems.

If you are experiencing any of the following, **changing the three knobs above will not help you**:

- **30-minute UI hangs with no error message.** My failures produce explicit `FailoverError` log lines at t=60 s. A silent hang that lasts half an hour is a different failure mode in a different layer.
- **Database save failures or conversation history corruption.** The idle timeout is an LLM-stream concern. Nothing in its code path writes or reads persistence.
- **OAuth disconnects, Google auth problems, or Anthropic provider bans.** Separate subsystems. (Anthropic's OAuth ban on OpenClaw from early April is unrelated to any of this.)
- **Problems running OpenClaw against cloud providers only.** Claude, OpenAI, Gemini do not have `NUM_PARALLEL=1` queues. If you only hit cloud endpoints and still time out, your bottleneck is elsewhere — likely in OpenClaw's scheduler itself, or in rate limits on the provider side.

My setup is local Ollama, four-agent fanout, M2 Max. That is the universe this post covers. Extrapolate at your own risk.

I have read the recent public complaints carefully, and at least one prominent case visibly involves cloud providers and UI-layer symptoms that do not match the signature I've documented here. That case is almost certainly *not* fixed by raising `idleTimeoutSeconds`. I am not claiming to solve it. I am documenting what my own logs prove.

---

## Closing: OpenClaw is not dead, but April was ugly

If you're on any version in the range **v2026.3.31 → v2026.4.13** and you run a local Ollama backend, there is a very high probability you are hitting the bug described above. The signature is:

- `FailoverError: LLM request timed out` with `durationMs` clustered near 62,000.
- Long serialized queues in Ollama's GIN log ending at the same second.
- Days that are clean, alternating with days that are catastrophic.

That signature matches [PR #55072](https://github.com/openclaw/openclaw/pull/55072) and the bisect in [issue #63200](https://github.com/openclaw/openclaw/issues/63200). v2026.4.14 ships [PR #66418](https://github.com/openclaw/openclaw/pull/66418) which addresses a related but separate layer (undici stream timeout propagation, closing [issue #63175](https://github.com/openclaw/openclaw/issues/63175)); the ongoing thread in [issue #67334](https://github.com/openclaw/openclaw/issues/67334) shows operators who upgraded to v2026.4.14 still hit timeouts even with `idleTimeoutSeconds` raised. Until the upstream picture finishes settling, the three-knob workaround is what I'm betting on — in the order described, with each knob pointed at a specific piece of the failure.

OpenClaw, as a project, is not dead. But April 2026 is going to go down as the month the out-of-the-box experience for local-Ollama users silently collapsed, and the main community signal for that collapse was a single high-visibility tweet thread that lumped together problems in several different layers.

If this post-mortem helps one person separate their queue-serialization timeout from someone else's DB-save bug, I'll consider it a win.

My logs, my setup, my diagnosis. Verify on your own box. Don't take my word for any of it.
