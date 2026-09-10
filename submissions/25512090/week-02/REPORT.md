# Week 02 Harness Comparison Report

## Part 1: Variant Definition

The two harnesses differ along the **five axes of agent architecture** introduced in the lecture:

| Axis | ReAct (`harness_react.py`) | Plan-then-Execute (`harness_plan_execute.py`) |
|------|----------------------------|-----------------------------------------------|
| **1. Context** | Full conversation history sent on every model call (`Chat` with `tools=True` by default) | Two separate conversations: planner (no tools, history = task + available tools) and executor (tools enabled, history = task + plan + step-by-step execution trace) |
| **2. Granularity** | Single tool call per step; the model decides the next action after each observation | Planner emits the entire plan (JSON list) in one call; executor runs multiple tool calls per step until the step is satisfied or budget exhausted |
| **3. Termination** | Iteration cap (`max_steps=8`); stops when model emits no tool calls | Fixed plan length + optional one replan (`max_replan=1`); executor stops after all steps complete, then a final answer call |
| **4. Error Handling** | Tool errors returned as observations; model decides how to react | Step-level `OFF_PLAN` signal triggers at most one replan; tool errors inside a step are observed and the model may continue or emit `OFF_PLAN` |
| **5. Intervention** | Human approval gate for `IRREVERSIBLE` tool calls (`interventions` counter) | No human-in-the-loop gate; flexibility limited to the single replan budget |

**Key behavioral difference**: ReAct interleaves reasoning and action at every step, allowing the model to adapt dynamically. Plan-then-Execute commits to a full plan upfront, then executes it rigidly with at most one course correction.

## Part 2: Measurements

| run | harness | success | tokens | iters | interventions | note |
|-----|---------|---------|--------|-------|---------------|------|
| 1 | react | X |  |  |  | crash: OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable |
| 2 | react | X |  |  |  | crash: OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable |
| 3 | react | X |  |  |  | crash: OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable |
| 4 | plan_exec | X |  |  |  | crash: OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable |
| 5 | plan_exec | X |  |  |  | crash: OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable |
| 6 | plan_exec | X |  |  |  | crash: OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable |

All six runs failed due to a missing `OPENAI_API_KEY` in the execution environment. The configured provider was OpenAI-compatible (Gemini via `generativelanguage.googleapis.com`), but no API key was available.

## Part 3: Interpretation

Since every run crashed before any model interaction, no meaningful token counts, iteration counts, or intervention data were collected. Both harnesses show identical failure modes (100% crash rate, 0% success), making comparative analysis impossible. To obtain valid measurements, a valid `OPENAI_API_KEY` with access to the Gemini 3.5 Flash Lite model must be supplied in the environment before re-running `run_ab.py --runs 3`.