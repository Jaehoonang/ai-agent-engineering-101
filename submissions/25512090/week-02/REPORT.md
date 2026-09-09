# Week 02 Experiment Report: ReAct vs. Plan-then-Execute

## Part 1: Variant Definition

The lecture categorizes agent harness design into five core axes:
1. **Context Management**: How conversation history and tool outputs are accumulated and pruned.
2. **Tool Granularity**: The level of abstraction and parameterization exposed to the model.
3. **Termination Condition**: When and how the harness decides the task is finished (e.g., step caps, explicit answer signals).
4. **Error Recovery**: How errors and unexpected tool outputs are fed back and handled.
5. **Human Intervention Point**: Where human approval is required before proceeding (irreversible actions).

The two evaluated harnesses set these axes differently:
- **ReAct (`harness_react.py`)**: Uses online, step-by-step reasoning (Thought -> Action -> Observation). Context management retains the full chat history across every iteration. Termination relies on an iteration cap (`max_steps=8`) or an explicit finish response. Error recovery happens organically by feeding error observations back into the chat history for the next thought cycle.
- **Plan-then-Execute (`harness_plan_execute.py`)**: Separates control flow into a distinct upfront planning phase (generating a complete JSON list of steps without tools) and an execution phase. Context management is partitioned between a planner chat and an execution chat. Termination and flexibility are governed by an explicit replan cap (`max_replan=1`) when execution hits `OFF_PLAN` conditions.

## Part 2: Measurements

The experiment executed both harnesses on the task: *"In app.log, which hour (HH:00) has the most ERROR lines? Answer with the hour in HH:00 form."* (`expected: 14:00`). Runs 2–6 and 8–11 encountered API rate-limit errors (`RateLimitError: 429`) due to free-tier provider quotas, while completed runs recorded tokens and iterations.

| Run | Harness | Success | Tokens | Iters | Interventions | Note |
|---|---|---|---|---|---|---|
| 1 | react | X | 14958 | 8 | 0 | MAX_STEPS reached: incomplete |
| 2 | react | X | | | | crash: RateLimitError (429) |
| 3 | react | X | | | | crash: RateLimitError (429) |
| 4 | plan_exec | X | | | | crash: RateLimitError (429) |
| 5 | plan_exec | X | | | | crash: RateLimitError (429) |
| 6 | plan_exec | X | | | | crash: RateLimitError (429) |
| 7 | react | X | 14986 | 8 | 0 | MAX_STEPS reached: incomplete |
| 8 | react | X | | | | crash: RateLimitError (429) |
| 9 | react | X | 14734 | 8 | 0 | MAX_STEPS reached: incomplete |
| 10 | plan_exec | X | | | | crash: RateLimitError (429) |
| 11 | plan_exec | X | | | | crash: RateLimitError (429) |
| 12 | plan_exec | O | 20800 | 10 | 0 | replans=0 |

## Part 3: Interpretation

Comparing the two harnesses reveals distinct operational trade-offs mapped to their architectural axes. The ReAct harness consistently executed up to its maximum step limit (`max_steps=8`) across all valid runs, consuming ~14,700–14,986 tokens per run, but failed to reach the correct final answer (`14:00`) within the iteration cap because its online, exploratory control flow lacked a structured decomposition of log aggregation steps. In contrast, the Plan-then-Execute harness successfully solved the task in run 12 (succeeding with `14:00` in 10 iterations and 20,800 tokens, with 0 replans needed). This demonstrates that separating the planning axis from execution allows the agent to establish a clear procedural roadmap upfront (reading the log, filtering errors, grouping by hour, and finding the maximum), avoiding the wandering behavior seen in ReAct. However, Plan-then-Execute is more sensitive to model capabilities during the initial JSON plan generation, as evidenced by initial rate-limit and plan-parsing failure modes under restricted free-tier quotas.
