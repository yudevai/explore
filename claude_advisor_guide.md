# Claude Advisor Tool

*Opus-level quality. Haiku-level cost. One API call.*

---

## Beta Notice

> **Beta feature.** Add the beta header to every request. API surface may change before GA.
> No waitlist — accessible via the header.
> Contact your Anthropic account team for feedback.

Beta header: `advisor-tool-2026-03-01`

---

## What It Is

Two Claude models on one task. The **executor** (Haiku or Sonnet) runs the full agentic loop — all tool calls, all output generation. When it hits a decision it can't confidently resolve, it calls the **advisor** (Opus 4.7) server-side. Opus reads the full context, produces a short plan (400–700 tokens), and the executor continues.

All of this happens inside a single `/v1/messages` call. No second request. No context synchronization. No orchestration layer. The executor decides when to call the advisor, just like any other tool.

---

## How It Works

1. You send a task to the executor model (Haiku or Sonnet).
2. The executor runs the agentic loop normally, calling tools as needed.
3. When it hits a complex decision, it emits a `server_tool_use` block — Anthropic handles the advisor call server-side.
4. Opus 4.7 reads the full conversation (system prompt, all turns, all tool results) and returns a plan. The executor continues with that guidance.

Opus only activates when the executor calls it. All bulk generation happens at the executor's rates. Advisor tokens are billed at Opus rates and reported separately in the `usage` block.

---

## Valid Executor / Advisor Pairs

- **Haiku 4.5 + Opus 4.7** — biggest quality lift, still far cheaper than Sonnet
- **Sonnet 4.6 + Opus 4.7** — near-Opus quality at similar or lower cost than Sonnet alone

Rule from the official docs: the advisor must be at least as capable as the executor. In practice, Opus 4.7 is the intended advisor model.

---

## Benchmarks — Official Numbers

| Executor    | Advisor    | Benchmark                  | Result                                      |
|-------------|------------|----------------------------|---------------------------------------------|
| Sonnet 4.6  | Opus 4.6   | SWE-bench Multilingual     | 74.8% vs 72.1% solo — 11.9% cheaper than Opus alone |
| Haiku 4.5   | Opus 4.6   | BrowseComp                 | 41.2% vs 19.7% solo (2x improvement)        |

> These benchmarks used Opus 4.6 as the advisor. With Opus 4.7 as the advisor the quality ceiling is higher.

---

## When to Use It

**Good fit:**

- Long overnight builds — most turns are mechanical, plan matters
- Multi-step agentic pipelines
- Computer use
- Complex research tasks
- Any workflow where you'd normally use Opus but most turns don't need it

**Not worth it:**

- Single-turn Q&A — nothing to plan, advisor never helps
- Simple prompts where every turn already resolves cleanly
- Tasks where every single turn genuinely requires Opus-level reasoning

---

## Setup — Python

Add the beta header and include the advisor tool in your `tools` array. The top-level `model` is the executor; `tools[].model` is the advisor.

```python
import anthropic

client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-haiku-4-5",             # executor
    max_tokens=4096,
    betas=["advisor-tool-2026-03-01"],    # required beta header
    tools=[
        {
            "type": "advisor_20260301",
            "name": "advisor",
            "model": "claude-opus-4-7",   # advisor
            "max_uses": 3,                # cap advisor calls per request
        },
        # ... your other tools here
    ],
    messages=[{
        "role": "user",
        "content": "Your task here"
    }]
)
```

---

## Multi-Turn Warning

In multi-turn conversations, you **must** include `advisor_tool_result` blocks verbatim in your message history. If you omit them, the API returns a `400 invalid_request_error`. Always append the full `response.content` to your messages array before sending the next turn.

```python
# Correct multi-turn pattern:
messages.append({
    "role": "assistant",
    "content": response.content  # include everything, including advisor blocks
})
messages.append({
    "role": "user",
    "content": "Your follow-up"
})

# Then send the next request with the same tools array.
# Removing the advisor tool from tools on a follow-up also causes a 400.
```

---

## Cost Reality

You pay each model's standard per-token rate for what it generates. The advisor generates only 400–700 tokens per consultation. The executor handles all bulk output at its lower rate.

- **Haiku + Opus advisor** — higher cost than Haiku alone, but far lower than running Sonnet or Opus end-to-end, with near-Sonnet quality
- **Sonnet + Opus advisor** — similar or lower total cost than Sonnet alone (confirmed from official docs), with a meaningful quality lift

Use `max_uses` to cap advisor calls per request — the primary cost control lever.

Advisor tokens appear separately in the `usage` block for clean cost attribution. A typical 25-turn coding agent with 3 advisor calls costs roughly **73% less** than running Opus end-to-end.
