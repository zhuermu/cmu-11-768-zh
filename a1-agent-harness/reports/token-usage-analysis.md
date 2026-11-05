# Token usage: compaction vs. no compaction

Instance: `django__django-15368`. Model: `deepseek-flash` (DeepSeek V4.1 Flash, OpenAI-compatible API).
Compaction threshold: 6000 estimated prompt tokens, keeping the most recent steps verbatim.
One sample per condition, so step counts are noisy; the context-size trends are not.

| | Compaction | No compaction |
|---|---:|---:|
| Resolved (`check-swebench`) | yes (1/1 F2P, 29/29 P2P) | yes (1/1 F2P, 29/29 P2P) |
| ReAct steps (action requests) | 22 | 16 |
| Compactions | 2 (steps 12 and 18) | 0 |
| Largest action prompt | **5,940** | **9,752** |
| Action prompt tokens, summed | 93,682 | 102,072 |
| Compaction-call prompt tokens | 10,199 | 0 |
| Completion tokens (actions + summaries) | 3,438 + 2,690 = 6,128 | 3,001 |
| **Total tokens billed** | **~110,000** | **~105,100** |
| Provider cache hits on input | 83,584 (81%) | 95,232 (93%) |
| Uncached input tokens | ~20,300 | ~6,840 |

Both runs produced byte-identical patches.

## What the numbers show

**Active context is bounded with compaction and grows linearly without it.**
Without compaction the prompt grows by roughly 500 tokens per step, from 1,701 to 9,752.
With compaction it climbs to the threshold, drops to about 3,000 after each summary
(5,889 → 3,091 at step 12, 5,940 → 3,018 at step 18), then climbs again: a sawtooth that never exceeds 6,000.
The summaries replaced ~5,900-token prompts with ~2,070-token ones (about 65% smaller).

**Total tokens did not go down on this task.** The task is short enough that the no-compaction
run never got near any context limit. Compaction added two summarization calls
(~10k input, ~2.7k output) and the compacted run took six more steps, so its total was about 5% higher.

**Compaction defeats prefix caching.** Each action request normally shares a long prefix with the
previous one, so the provider serves most input from cache (93% without compaction). Rewriting the
prefix into a summary invalidates that cache, and the summarization calls themselves are almost entirely
cold. Uncached input, which is what dominates cost and latency on a cached provider, was about 3x higher
with compaction.

**Extra steps after compaction.** Right after the first summary the agent re-read material it had
already seen: step 13 re-opened `query.py` lines 660-685 alongside `git diff`, and step 15 re-read
`tests/queries/test_bulk_update.py` lines 1-30 and 190-225, nearly the same ranges as step 5. After the
second summary, step 19 re-ran `git diff` and the tests. The summary preserved *what* was done but not
the exact text the agent had relied on, so it fetched it again.

## Tradeoffs

- Compaction buys a hard ceiling on context size. That matters for long tasks that would otherwise exceed
  the model's window, and for models whose quality degrades on long contexts. On a 16-step task with a
  10k-token peak it bought nothing.
- It costs extra model calls, lost cache reuse, and lossy memory: details that were not written into the
  summary must be rediscovered, which shows up as extra steps.
- A threshold this low (6000) compacts often. In practice the threshold should sit well above the typical
  task length and close enough to the real context limit that compaction is a safety net, not a routine step.
