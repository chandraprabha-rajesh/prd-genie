# PRD Genie — Baseline Dataset Testing

## Method

Fourteen test inputs were run against the pipeline. Each has a defined expected
behaviour and explicit pass/fail criteria written **before** the run, so that a
plausible-looking output could still be recorded as a failure.

Most tests target Agent 1 (Requirement Extractor), because every later agent consumes
its output — an error there propagates. T11 and T12 target the PRD Generator and Story
Breakdown. T13 targets the refusal path through the human checkpoint.

**Tests were run once, in the Automated Chain flow.** Agent 1's prompt is identical in
both flows, so running the extraction tests twice would produce no additional
information. Tests that exercise the human checkpoint (T13, and the T4 correction
demonstration below) were run in the Human-in-the-Loop flow.

All agents ran at temperature 0 on gpt-4o-mini, with Langfuse tracing enabled. The
Grounding evaluator scored every generation automatically as the tests ran.

---

## Results

| ID | What it tests | Result |
|---|---|---|
| T1 | Baseline extraction from a detailed transcript | PASS |
| T2 | Refuses to invent from a vague brief | PASS |
| T3 | Records contradictions without resolving them | PASS |
| T4 | Preserves acceptance criteria exactly | PASS (observation) |
| T5 | Does not fill gaps in incomplete notes | PASS |
| T6 | Keeps multiple stakeholder positions distinct | **FAIL** |
| T7 | Preserves technical values character for character | PASS |
| T8 | Keeps personas separate | PASS |
| T9 | Stops when there is nothing to extract | PASS |
| T10 | Flags dependencies and unknowns | PASS |
| T11 | PRD contains only what the extraction contained | PASS |
| T12 | Story breakdown adds no scope | PASS |
| T13 | Refuses to generate a PRD from an empty extraction, even when instructed to proceed | PASS |
| T14 | Guards do not block valid input (regression) | PASS |

**13 pass, 1 fail.**

---

## Detail — tests run 20 November

### T3 — Contradictions
**Input:** "The dashboard should auto-refresh every 5 seconds. Later in the discussion: Performance is critical, minimize API calls."

**PASS.** Recorded both positions, identified the conflict, did not pick a side, did
not merge them, did not advise which was better. The contradiction was carried
forward into the PRD's Open Questions and into the Gap Analyzer's Unresolved
decisions.

**Caveat on this test's strength:** this scenario appears verbatim in Agent 1's own
prompt as Example 3. A pass demonstrates that the example is followed; it does not
demonstrate that unseen contradictions are detected. T6 tested that, and failed.

### T4 — Acceptance criteria
**Input:** "Users need to export reports as PDF and CSV. PDF must include company logo. CSV must preserve formulas."

**PASS with observation.** Both criteria were extracted verbatim and nothing was
added.

However, the source stated the criteria without labelling them as criteria — which is
how real discussions work. Agent 1 recorded all four statements as requirements.
Downstream, the PRD's Acceptance Criteria section read "Not discussed" despite the
source containing two, and the Story Breakdown produced "include the company logo" as
a user story of its own. The Gap Analyzer then asked *"What are the acceptance
criteria for including the company logo in the PDF export?"* — a circular question
about an item that was itself a criterion.

This is not a prompt defect. Distinguishing a requirement from its criterion is an
inference the source did not make, and Agent 1 is forbidden from inferring. The
correction belongs at the human checkpoint — see the demonstration below.

### T5 — Incomplete notes
**Input:** "John mentioned a dashboard. Real-time. Budget TBD."

**PASS.** Stated Requirements: none. Flagged "real-time" and "budget TBD" as
ambiguous, listed three missing-information questions, invented nothing.

The refusal then propagated correctly through the whole chain: the PRD Generator
declined to generate, and the Story Breakdown, Gap Analyzer and Scope Estimator each
declined in turn. This was the first end-to-end refusal in the Automated Chain —
previously only tested through the human-in-the-loop flow.

Run time 9.7s against ~25s for a full generation, because four agents produced two
lines each instead of a document.

### T6 — Multiple stakeholders — **FAIL**
**Input:** "Engineering wants microservices. Design wants single-page app. PM wants it shipped by March."

All three positions were captured and kept distinct, and none was favoured. But the
extractor then wrote:

```
Contradictions
Engineering wants microservices, while Design wants a single-page app.
Both were stated. The trade-off was not resolved.
```

**There is no contradiction.** Microservices is a backend decomposition; a single-page
app is a frontend approach. The two are routinely used together. The output asserts a
trade-off and never names what the trade-off is.

This is the same failure as the Gap Analyzer's fabricated stakeholder debate found
earlier in the build: the agent invents a *relationship* between two statements that
the source never connected. T8 produced the mirror image — an invented agreement.

Recorded as defect D1.

### T7 — Technical values
**Input:** "API must support 10,000 concurrent users. Response time < 200ms at p95. Must integrate with Salesforce REST API v52."

**PASS**, and the strictest test in the set.

| Check | Result |
|---|---|
| `10,000` not rounded to "10K" | ✅ |
| `< 200ms at p95` preserved exactly | ✅ |
| `v52` not altered to "v5.2" | ✅ |
| Performance values classified as NFRs | ✅ |
| `p95` not explained from the model's own knowledge | ✅ — listed under Unexplained Terms |

The last is the most informative. The model knows what p95 means and did not say,
because the input did not define it.

### T8 — Personas
**Input:** "Admins need bulk user management. End users need a simplified view. Auditors need read-only access with full history."

**PASS.** Three personas kept distinct through the whole chain — the PRD's User
Personas section was populated (it reads "Not discussed" on inputs that name no
personas), and the stories read "As an Admin", "As an End User", "As an Auditor".

Two observations, recorded as D10 and D11:
- `Ambiguous Items: None.` although "a simplified view" and "full history" are
  undefined. The Gap Analyzer caught all three downstream and marked them blocking.
  The same agent correctly flagged "better reporting" in T2, so the behaviour is
  inconsistent — a vague term inside a complete sentence reads as specified.
- `DECIDED: The requirements for admins, end users, and auditors were agreed upon.`
  Nobody agreed anything. Three stated needs became a recorded decision.

### T10 — Dependencies
**Input:** "SSO login requires the new auth service which is being built by Team Alpha. ETA unknown."

**PASS.** Extracted the SSO requirement, named Team Alpha, carried the unknown ETA
through as unresolved, and did not estimate a date or treat the dependency as closed.

One column error: the Dependencies table recorded *"ETA for the new auth service is
unknown"* as the **dependency**, with Risk left as "Not stated". The dependency is the
auth service; the unknown ETA is the risk. The two were swapped.

---

## Demonstration — what the human checkpoint adds

T4 was re-run through the Human-in-the-Loop flow with one PM correction at the
checkpoint:

> "The PDF export must include the company logo" is not a separate requirement — it is
> an acceptance criterion of the PDF export requirement. [same for the CSV formulas]

| | Automated | With one PM correction |
|---|---|---|
| Functional requirements | 4 | **2** |
| PRD Acceptance Criteria section | "Not discussed." | **both criteria present** |
| Priority column | all "Not stated" | all "Not stated" |
| Gap Analyzer questions | 8 | **4** |
| — of which circular | **2** | **0** |
| Gap Analyzer tokens | 9.4K | **7.0K** |
| Gap Analyzer latency | 34.8s | **19.7s** |

Two sentences from a reviewer corrected a structural error that no prompt revision had
been able to prevent — because the error was not a mistake in following instructions,
but a judgement the source material did not contain.

### The clearest single piece of evidence

The same agent, on the same source document, with the same prompt, asked these two
questions in the two runs:

> **Automated:** "What are the acceptance criteria for including the company logo in
> the PDF export?"
>
> **Human in the Loop:** "What specific details should be included in the PDF export
> beyond the company logo?"

The first asks for the acceptance criteria *of an acceptance criterion*. It is
unanswerable, and it is marked BLOCKING, so it would stop work on a story that is not
actually blocked. The second treats the logo as settled and asks what else is needed,
which is a question a PM can act on.

The error originated at Agent 1 and was still doing damage at Agent 4, three hops
later. Nothing in between caught it, because at every hop it was being handled
correctly — Agent 2 formatted what it was given, Agent 3 wrote stories from what it
was given, Agent 4 found gaps in what it was given. Each agent obeyed its
instructions. The mistake was upstream of all of them.

### The correction made the system cheaper, not more expensive

| | Automated | With correction | Change |
|---|---|---|---|
| Gap Analyzer tokens | 9.4K | 7.0K | **−26%** |
| Gap Analyzer latency | 34.8s | 19.7s | **−43%** |

This is the opposite of the expected trade-off. A review step is normally justified
as quality bought at the price of throughput. Here two sentences of human input
reduced downstream token spend and latency at the same time as improving the output,
because four of the automated run's eight questions were about items that were
already answers. The agent was spending tokens interrogating its own input error.

It also changed *which* questions got asked. The human-in-the-loop run asks about
performance requirements for the export and which user personas are relevant —
genuine gaps. The automated run never reaches them. The noise crowded out the signal.

Two details worth noting. The PRD's Source Traceability line recorded
`Acceptance Criteria: Derived from PM feedback on Stated Requirements 3 and 4` —
the system attributed the correction to the human rather than to the extraction,
unprompted. And the Scope Estimator wrote *"the sizing is reliable given the quality
of the source material"* — a phrase it had earlier been corrected for using falsely.
Here it was true. The guardrail stopped the claim when it was unfounded without
preventing it when it was warranted.

---

## Evaluator scores

Three evaluators scored every generation in the project, applied retrospectively to
the full trace history as well as to new runs.

| Evaluator | Type | Question it asks | Observations | Mean | Std dev |
|---|---|---|---|---|---|
| Grounding | LLM judge | Did the output assert something the source did not say? | 258 | **0.66** | 0.48 |
| Coverage | LLM judge | Did something the source did say fail to reach the output? | 518 | **0.29** | 0.45 |
| Overstated Confidence | Code, no model | Does the output claim its source was more complete than it was? | 259 | **0.98** | 0.15 |

In all three, 1 is the good outcome. A judge may score 0 only if it can quote the
specific offending line; where it cannot quote one, it must score 1. This makes the
scores conservative — a real failure the judge cannot evidence is recorded as a pass.

### Reading these numbers

**Grounding at 0.66** means roughly one output in three contains a claim that is not
in its source. This supports the architecture rather than undermining it: the system
is positioned as a drafting assistant with a human checkpoint, not as an authority,
and this is the number that justifies that position.

**Coverage at 0.29 is the finding of the project.** Seven outputs in ten lose
something the source contained. Every rule in all five prompts was written to prevent
invention — *use only what is stated, never infer, write UNKNOWN* — and invention
turns out to be less than half as common as loss. The effort went almost entirely
into the smaller of the two failure modes.

The two evaluators read the same traceability requirement in opposite directions.
Grounding asks whether every line of output traces back to the source. Coverage asks
whether every item in the source reaches the output. Only the first was being
designed for.

**Overstated Confidence at 0.98** is high because it is a string check against a
short phrase list, not a judgement. It catches the specific failure it was built for
— the Scope Estimator claiming a source was "fully specified" when it was not — and
nothing else. It is included as the cheap deterministic counterpart to the two
judges, and it costs nothing to run because no model is involved.

### Caveat on the Coverage population

Coverage ran on every generation in the pipeline, not only on ingestion. An
observation therefore measures loss at whichever hop it scored — extraction, PRD
generation, story breakdown, gap analysis or sizing — which is why it has roughly
twice as many observations as the other two. The 0.29 is loss *per hop*, not loss
between the source document and the final output. This was not the original design:
the rule filter could not be scoped to the first agent, and measuring every hop
turned out to be the more useful accident.

---

## What the test run revealed

**The tests found what review by reading did not.** The grounding failures in T6 and
T8 — an invented contradiction and an invented agreement — both read as competent
output. Nothing about them looks wrong until the output is compared line by line
against a three-sentence input.

**One failure class dominates.** Of eleven defects recorded, the most serious (D1, D11)
are the same error: the agent inventing a relationship between statements that the
source never connected. Priority, product names, acceptance criteria and risk ratings
had already been stopped individually; this is the same behaviour reaching for the one
remaining shape.

**The tests designed from the prompt's own examples are the weakest.** T3 passes
because its scenario is in the prompt. T6 fails because its scenario is not. Any
future test set should be written before the prompts, not after.
