+++
date = '2026-05-09T01:20:40-04:00'
draft = false
title = 'Building a Multi-Turn LLM Tool-Calling Pipeline'
tags = ['agentic AI', 'LLM']
+++


If you've used the OpenAI, Anthropic, or Bedrock APIs to build something more sophisticated than a chatbot, you've probably written an agent loop — code that lets the model call tools, receive results, and decide what to do next. I recently built one for a document analysis pipeline at work, and a few things surprised me. This post is a distillation of those lessons, using a generic example.

## The Setup: A Four-Tool Pipeline

Imagine you're processing a document. For each item the model identifies, you want to:

1. Run a fast keyword search against a reference database
2. For items that don't match, run a semantic vector search
3. Apply some classification logic to the results
4. Save the findings and emit a summary

Four tools, in order. The model orchestrates them.

```
keyword_search → vector_search → save_results → emit_summary
```

## Lesson 1: The API Is Stateless. Your Code Manages Everything.

The first thing that surprised me: every API call is completely independent. The model has no memory of previous calls. Your code has to send the entire conversation history — system prompt, tool definitions, and every prior message — on every single call.

The lifecycle of a five-call pipeline looks like this:

```
Call 1:  system + tools + [user]                                       → tool_use (keyword_search)
Call 2:  system + tools + [user, asst, TR]                             → tool_use (vector_search)
Call 3:  system + tools + [user, asst, TR, asst, TR]                   → tool_use (save_results)
Call 4:  system + tools + [user, asst, TR, asst, TR, asst, TR]         → tool_use (emit_summary)
Call 5:  system + tools + [user, asst, TR, asst, TR, asst, TR, asst, TR] → end_turn
```

The system prompt and tool definitions are constant across all five calls. The messages array grows by two entries per turn — the assistant's response and the tool result.

This is why prompt caching exists. If you're sending the same 8,000-token system prompt five times, you're paying to re-process it five times. Anthropic and Bedrock both let you mark prefixes as cacheable, which can dramatically reduce cost and latency for multi-turn flows.

## Lesson 2: The Model Is the Router

This was the most clarifying realization for me. I initially thought my tools needed routing logic — "if keyword search hits, skip vector search." But that's not how it works.

The model sees the keyword search result and decides what to send to the next tool. If three out of ten items got hits, it sends only the seven non-hits to vector search. If all ten got hits, it skips vector search entirely. Your tools don't need any conditional logic — they just execute what they're called with.

```
Tool 1 returns: [item-1: hit, item-2: no, item-3: hit, item-4: no]
                                ↓
Model constructs Tool 2 input: [item-2, item-4]   ← only the misses
```

Your tools should be dumb executors. The model is the brain.

## Lesson 3: Use Stable IDs as Passthrough Fields

Here's a subtle trap. Suppose your keyword_search tool just takes an array of strings and returns an array of results in the same order. The model has to mentally track "result[0] corresponds to query[0], result[1] to query[1]" across multiple turns.

This works most of the time — but it's fragile. The model is doing positional bookkeeping in its head, across multiple context-heavy turns, with no guardrails.

A better pattern: include a stable ID as a passthrough field that the tool echoes back without using internally.

```json
// Input
{ "candidates": [
    { "item_id": "1-1", "text": "..." },
    { "item_id": "1-2", "text": "..." }
]}

// Output — item_id echoed back
{ "results": [
    { "item_id": "1-1", "match": true, "match_id": "REF-482" },
    { "item_id": "1-2", "match": false }
]}
```

The tool doesn't need to know what `item_id` means. It's just a tracking key. But now the model can reliably correlate results back to the original items, even across multiple turns and tools. The cost is trivial; the reliability gain is significant.

## Lesson 4: Batch Writes at the End Beat Per-Tool Writes

I had to choose between writing results to my database after each tool call, or batching everything and writing once at the end. I chose batch.

```
Per-tool writes (rejected):
  Tool 1 writes 3 records, Tool 2 writes 7 records, Tool 3 fails
  → 10 records in DB from a failed run, need cleanup logic

Batch write (chosen):
  Tool 1 returns to model, Tool 2 returns to model, Tool 3 writes all 10
  → If Tool 3 fails, nothing in DB, clean retry
```

The batch approach gives you atomicity for free. Either the whole pipeline lands in the DB or nothing does. With per-tool writes, you have to handle partial state, deduplication, and rollback — meaningful complexity for a marginal latency gain.

The exception: if your pipeline has many turns and you can't afford to lose intermediate work, checkpointing makes sense. But for short, deterministic pipelines, batch is simpler and safer.

Make sure your terminal tool is idempotent — a unique constraint on (run_id, item_id) handles the retry case cleanly.

## Lesson 5: The Model Decides When to Stop

The agent loop's exit condition isn't something your code computes. It's whatever the model returns.

```python
while turn_count < max_turns:
    response = call_api(system, tools, messages)
    if response.stop_reason == "end_turn":
        break  # model has nothing left to do
    if response.stop_reason == "tool_use":
        execute_tool(response)
        append_to_messages(response)
        # loop continues
```

The model "decides" to stop by simply generating text instead of a tool call. There's no explicit completion classifier. It's next-token prediction — the model has been trained to recognize when it has enough information to answer, and at that point the most probable next tokens form a text response rather than a tool call.

Two consequences:

- **Your prompt shapes the stop behavior.** If you tell the model "be thorough — verify with at least three sources," it'll keep calling tools longer. If you say "stop as soon as you have a definitive answer," it stops sooner. You're shifting probabilities at the decision point.
- **You need a `max_turns` safety net.** If the model gets stuck in a loop or your prompt is ambiguous, you don't want infinite tool calls. A hard limit protects you.

## The Trade-off Between Raw API and Higher-Level SDKs

You can write this loop yourself against the raw Anthropic, OpenAI, or Bedrock API. Or you can use a higher-level SDK (like the Claude Agent SDK or LangChain) that handles the loop for you.

The raw approach gives you control. You see every message, every tool call, every retry. You can shape the loop to fit your domain — custom retry logic, specialized error handling, fine-grained observability.

The SDK approach gives you speed. You skip the boilerplate and get features like context compaction (when conversations exceed the window) and automatic budget enforcement.

For my pipeline — short, deterministic, four tools — raw API was the right choice. For a long-running agent that needs to chain dozens of operations across hours, the SDK earns its weight.

## Summary

If I had to compress this post into a list:

- Every API call is stateless. Your code sends everything every time.
- The model orchestrates tools. Your tools should be dumb executors.
- Stable IDs as passthrough fields beat positional inference.
- Batch your writes at the end. Atomicity beats complexity.
- The model decides when to stop. Shape it through prompting; protect it with `max_turns`.
- Use prompt caching for multi-turn flows. The system prompt is a tax otherwise.

Tool calling looks like magic from the outside. Up close, it's just a structured loop with the model in the driver's seat — and once you internalize that, designing these pipelines becomes a lot more straightforward.

---

**A few notes on this draft:**

I deliberately kept the tool names generic (`keyword_search`, `vector_search`) — they map cleanly to the two-stage retrieval pattern that's common in RAG and search systems, so readers from many domains will recognize the shape without you giving away your specific use case.

I'd suggest **not** including the full JSON message exchange in the published version — it adds length without adding much beyond what the prose already conveys. If you do want JSON in there, a single short example (like the `item_id` passthrough one) is enough to make the point.

The "Trade-off Between Raw API and Higher-Level SDKs" section is optional — cut it if you want a tighter post focused purely on the loop mechanics.

Want me to adjust the tone (more technical / more accessible), tighten the length, or rework any specific section?