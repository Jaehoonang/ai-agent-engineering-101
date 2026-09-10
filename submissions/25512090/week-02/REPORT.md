# Week 02 Report: ReAct vs. Plan-then-Execute Harness Comparison

## Part 1: Variant Definition

The five design axes of agent harnesses are:
1. Context management
2. Tool granularity
3. Termination condition
4. Error recovery
5. Human intervention point

**ReAct (`harness_react.py`)** and **Plan-then-Execute (`harness_plan_execute.py`)** differ primarily across **Error Recovery**, **Context Management**, and **Termination Condition** (execution flow):
- **ReAct** interleaves thought, action, and observation dynamically at each step. It adapts its next step based on the immediate output of the previous tool call, allowing runtime error recovery and dynamic termination when the answer is reached or max steps are hit.
- **Plan-then-Execute** separates planning from execution entirely. It prompts the model to output an entire sequence of tool steps upfront as a JSON list before executing them sequentially. If the model fails to produce valid JSON or encounters intermediate errors, it lacks dynamic re-planning mechanisms during the execution phase.

---

## Part 2: Measurements

| Run | Harness | Success | Tokens | Iters | Interventions | Note |
|---|---|---|---|---|---|---|
| 1 | react | X | 14734 | 8 | 0 | Max steps reached |
| 2 | react | X | - | - | - | RateLimitError: 429 Quota exceeded |
| 3 | react | X | - | - | - | RateLimitError: 429 Quota exceeded |
| 4 | plan_exec | X | - | - | - | RateLimitError: 429 Quota exceeded |
| 5 | plan_exec | X | - | - | - | RateLimitError: 429 Quota exceeded |
| 6 | plan_exec | X | - | - | - | RateLimitError: 429 Quota exceeded |

---

## Part 3: Interpretation

Both harnesses achieved zero successes (X) due to model rate limits (HTTP 429 Quota Exceeded on the free tier of `gemini-3.5-flash-lite`) and step-budget limitations. In Run 1 (ReAct), the agent successfully invoked `read_file` and `count_pattern` across 8 iterations consuming 14,734 tokens, but exhausted `MAX_STEPS` before outputting the final answer in the required format. Subsequent runs (Runs 2–6) crashed immediately due to API quota exhaustion during rapid successive requests. This highlights how free-tier API rate limits and token budgets heavily influence harness reliability, and demonstrates that Plan-then-Execute and ReAct both depend heavily on robust model rate management and prompt-following capabilities for structured JSON output and multi-step reasoning.
