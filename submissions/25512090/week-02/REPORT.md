# Week 02 Harness Comparison Report

## Part 1: Variant Definition

The experiment compares two distinct agent execution harnesses—**ReAct** (`harness_react.py`) and **Plan-then-Execute** (`harness_plan_execute.py`)—across the five design axes of agent architectures:

1. **Context Management (맥락 관리)**
   - **ReAct**: Maintains a single continuous chat session (`Chat`) containing the entire conversation history (system instructions, user query, thoughts, tool calls, and observations) in every turn.
   - **Plan-then-Execute**: Separates context into two distinct roles: a stateless **Planner** (`SYSTEM_PLAN`) that creates a JSON step list without tools, and an **Executor** (`SYSTEM_EXEC`) that maintains execution state and carries out individual plan steps with tools.

2. **Action Granularity (작동 입도)**
   - **ReAct**: Operates at a micro-step level. At each turn, the model dynamically generates a `Thought:` and selects the next tool action based on the immediately preceding observation.
   - **Plan-then-Execute**: Operates at a macro-step level. The planner first decomposes the overall task into a structured sequence of high-level plan steps, which the executor then processes sequentially.

3. **Termination Criteria (종료 조건)**
   - **ReAct**: Terminates either when the model outputs a final response without any tool calls (starting with `Answer:`) or when hitting the maximum iteration cap (`max_steps=8`).
   - **Plan-then-Execute**: Terminates when all steps in the plan array have been executed and a final response starting with `Answer:` is returned. Step-level termination is controlled by a sub-step tool budget (`max_tool_rounds=3`), and overall flexibility is capped by `max_replan=1`.

4. **Error Recovery (오류 복구)**
   - **ReAct**: Error recovery is implicit and reactive. Exceptions or tool errors are passed back as `Observation` messages, allowing the model to adjust its next thought and tool call dynamically.
   - **Plan-then-Execute**: Error recovery is explicit and structural. When a step fails or exceeds the tool-call budget, the executor returns an `OFF_PLAN:` signal. If `replans < max_replan`, the planner is re-invoked with failure feedback to generate new remaining steps.

5. **Human Intervention (인간 개입)**
   - Both harnesses share the same safety infrastructure: checking tools against an `IRREVERSIBLE` set before execution. In this experiment, read-only log tools were used (`IRREVERSIBLE = set()`), so intervention counts remained 0 across all runs.

---

## Part 2: Measurements

The experiment was executed with 3 runs per harness using model `gemini-3.5-flash-lite`. The measured metrics recorded in `results.csv` are as follows:

| run | harness | success | tokens | iters | interventions | note |
|---|---|---|---|---|---|---|
| 1 | react | X | 14650 | 8 | 0 | |
| 2 | react | X | 14706 | 8 | 0 | |
| 3 | react | X | 14986 | 8 | 0 | |
| 4 | plan_exec | O | 4937 | 4 | 0 | replans=0 |
| 5 | plan_exec | O | 14424 | 8 | 0 | replans=0 |
| 6 | plan_exec | O | 27433 | 15 | 0 | replans=1 |

---

## Part 3: Interpretation

The experimental results demonstrate a clear contrast in reliability between the two harnesses for this log analysis task. ReAct failed in all 3 runs (0/3 success rate), consistently reaching `MAX_STEPS` (8 iterations) because it redundantly queried hourly patterns one by one without a macro view, exhausting its step budget before producing a final answer. In contrast, Plan-then-Execute achieved a 100% success rate (3/3). By establishing a structured multi-step plan up front, the executor aggregated error counts efficiently; furthermore, in run 6 when step 1 exceeded the tool-call budget, the explicit `OFF_PLAN` mechanism triggered a replan (`replans=1`), allowing the agent to dynamically adjust its remaining steps and arrive at the correct expected answer (`14:00`).
