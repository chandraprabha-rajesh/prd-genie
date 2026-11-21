# Screenshots

Evidence for the claims in the top-level README, captured after the prompts were
frozen and both flows verified identical.

## The build

| File | What it shows |
|---|---|
| `01_flow_automated.png` | The Automated Chain in LangFlow. File → Extractor → Prompt Template → PRD Generator → Story Writer → Gap Analyser + Scope Estimator in parallel. |
| `02_flow_hitl.png` | The Human-in-the-Loop flow. Note the gap: the extractor feeds a Chat Output and nothing else, and a separate Chat Input feeds the Prompt Template. The two halves are not connected. |
| `03_guard_step1_prompt.png` | Agent 2's OUTPUT FORMAT, showing the STEP 1 input check. Four rule-based attempts failed to stop it generating from an empty extraction; moving the refusal into the output structure worked immediately. |

## The human checkpoint

| File | What it shows |
|---|---|
| `04_checkpoint_approve.png` | A reviewed extraction passed forward: *"Reviewed. Looks fine, go ahead with PRD creation."* |
| `05_checkpoint_reject.png` | A reviewer stopping the pipeline: *"Reviewed. No requirements here. Do not create a PRD."* |

## Observability

| File | What it shows |
|---|---|
| `06_langfuse_traces_list.png` | The full trace history. Every run is captured, not a selected sample. |
| `07_langfuse_trace_expanded.png` | One complete run as a span tree — all five agents with per-node latency, cost and inline evaluator scores. |
| `08_cost_full_run.png` | Full run: 50.43s, $0.004, 16k tokens. |
| `13_refusal_trace.png` | A guarded refusal: 14.52s, $0.001, 7k tokens. Cost falls faster than token count because what a refusal eliminates is output tokens, which cost roughly four times input tokens. |

## Evaluation

| File | What it shows |
|---|---|
| `09_evaluators_configured.png` | Three evaluators live — two LLM judges and one code evaluator that runs no model. |
| `10_score_grounding.png` | Grounding: 258 observations, mean **0.66**. Did the output assert what the source did not say? |
| `11_score_coverage.png` | Coverage: 518 observations, mean **0.29**. Did what the source did say fail to reach the output? |
| `12_score_overstated_confidence.png` | Overstated Confidence: 259 observations, mean **0.98**. Does the output claim its source was more complete than it was? |

In all three, 1 is the good outcome. Coverage at 0.29 against Grounding at 0.66 is the
finding of the project: every rule in all five prompts was written to prevent
invention, and omission turns out to be more than twice as common.

## What the checkpoint buys — test T4

The same source document through both flows.

| File | What it shows |
|---|---|
| `14_t4_prd_before_after.png` | Automated produces 4 functional requirements and "Not discussed." under Acceptance Criteria. With one two-sentence correction: 2 requirements, both criteria present. |
| `15_t4_gap_automated.png` | The Gap Analyser asks 8 questions, two of them unanswerable — including *"What are the acceptance criteria for including the company logo in the PDF export?"*, which asks for the acceptance criteria of an acceptance criterion, and marks it BLOCKING. 9.4K tokens, 34.8s. |
| `16_t4_gap_hitl.png` | The same agent, same prompt, after the correction: 4 questions, none circular, and it now asks about export performance and user personas — real gaps the automated run never reached. 7.0K tokens, 19.7s. |

The correction reduced downstream token spend by 26% and latency by 43% while
improving the output. A review step is normally justified as quality bought at the
cost of throughput; here it was both.
