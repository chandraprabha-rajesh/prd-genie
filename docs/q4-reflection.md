# PRD Genie — Q4: Reflection

## 1. What I got wrong initially, and what it cost me

**I created the test cases after writing the prompts.** I tested the workflow against 14 inputs, with 13 passing. Initially, I considered this a good result. However, when I looked more closely, I realised that the pass rate was not entirely reliable.

For example, T3 tested how the Requirement Extractor handled contradictions. The scenario used in this test was already present as Example 3 in the agent's prompt. It passed, but that did not necessarily mean the agent could identify contradictions in a new scenario.

T6 tested the same capability using a scenario that was not included in the prompt. This time, the extractor incorrectly identified a contradiction between microservices and a single-page application, even though the two are not mutually exclusive. It also failed to explain what the actual conflict was.

This made me realise that I had been testing whether the model could follow examples in its instructions rather than whether it could handle situations independently. I later froze the prompts and created three additional test inputs. I should have followed this approach from the beginning.

**I also focused too much on preventing hallucinations and overlooked information loss.** Most of my prompt instructions across the five agents were designed to prevent the model from inventing information: use only what is stated, do not infer, and return UNKNOWN when information is missing.

However, when I introduced a second evaluator to measure information coverage, the result was quite different. The Coverage score was 0.29, compared with 0.66 for Grounding.

In other words, I had spent most of the development effort preventing the system from adding information that was never discussed, without paying enough attention to whether it was retaining everything that was actually discussed.

I had considered traceability mainly in one direction: whether every output could be traced back to the source. I had not given equal importance to checking whether every relevant source item made it into the output.

## 2. What I learned about prompt and output design

One of the biggest learnings during implementation was that **changing the output structure was often more effective than adding more prompt instructions.**

For example, Agent 2 continued generating a complete PRD even when the extraction was empty. I tried adding four increasingly explicit instructions asking it not to proceed. None of them worked reliably.

Eventually, I moved the refusal instruction into the OUTPUT FORMAT section as a separate STEP 1 branch. That worked on the first attempt.

I saw similar issues in other agents:

- A performance requirement kept appearing under Stated Requirements. Moving the Non-Functional Requirements heading above it in the output format resolved the issue.
- The model kept assigning "Must" priority to everything. Adding "Not stated" as an available option in the template helped address this.
- Non-functional requirements were being converted into user stories. Introducing a separate Technical Tasks section gave the model a place to put them.

I also realised that the template itself influences the model's output. The PRD template contained placeholder rows such as `| NFR-001 | | Performance | |`. These categories appeared in the generated PRDs, even when the source had not mentioned them. I initially tried fixing this through prompt instructions, without realising that the template was introducing the same information I was asking the model not to invent.

I faced a similar issue with examples in the Scope Estimator prompt. Some examples demonstrated phrasing that I had explicitly prohibited elsewhere in the instructions.

My main learning here was that prompts cannot be maintained in isolation. The output format, examples and templates also influence model behaviour. Adding more instructions is not always the right fix; sometimes the better solution is to change the structure the model is expected to follow.

## 3. What surprised me during testing

**Getting an individual agent to behave correctly did not mean the complete workflow would behave correctly.**

One example was the empty extraction scenario. Agent 2 correctly refused to generate a PRD because there was no meaningful input. However, Agent 3 interpreted that refusal message as valid input and generated an authentication product with registration, login and password recovery features.

Agent 5 then assigned sizes to the invented stories, while Agent 4 generated gaps and assigned owners to them.

So, even though Agent 2 had behaved correctly, the overall workflow produced a complete plan for something that did not exist.

Until then, I had mainly been testing individual agents and assuming that if each agent worked correctly, the complete workflow would also work correctly. This test showed me why that assumption was wrong. In a sequential pipeline, an error or incorrect interpretation at one stage can affect everything that follows.

**Another unexpected finding was that human review actually reduced processing time and cost.**

I initially expected the human review step to improve quality at the expense of execution time. However, the results from T4 showed the opposite.

A two-sentence correction from the PM reduced the Gap Analyser's token usage by 26% and execution time by 43%. This happened because four of the eight questions generated by the automated workflow were based on an incorrect interpretation of the requirements.

For example, the automated workflow asked for acceptance criteria for something that was already an acceptance criterion and even marked it as BLOCKING.

Once the PM corrected the extraction, the downstream agent could focus on the actual missing information instead of questioning something that was already clear.

This changed how I looked at the human review step. It was not simply an additional approval activity. Correcting information early could also reduce unnecessary downstream processing.

**I also realised that having a human review step does not automatically mean the workflow is controlled.**

When the extraction was empty, the PM could respond with "looks fine, go ahead", and the workflow could still generate a PRD. I tried addressing this through the same four prompt revisions, but it was not reliable.

I learned that there is a difference between asking the model to respect an approval decision and actually controlling whether the workflow can proceed.

A PM correction belongs in the input because the model needs to read and apply it. But a go/no-go decision should be enforced through a workflow-level check. For a production implementation, I would add a validation step so that the workflow proceeds only when the extraction contains valid information and has been approved by the PM.

**The evaluator itself also had a similar problem.**

My Grounding evaluator sometimes reported that an output contained several invented claims but did not quote any specific example to support its score. I initially tried fixing this by adding more instructions to the evaluator prompt. That did not solve the issue.

Eventually, I introduced a simple structural condition: if the evaluator assigns a score of 0, it must quote the exact output line. If it cannot provide a supporting quote, the score is 1.

This worked better. It also reinforced something I had already observed with the agents: where a check can be expressed as a simple rule, using code is often more reliable than asking another LLM to make the decision.

## 4. Mistakes I made during the build

There were also two conclusions I accepted during analysis that turned out to be wrong.

- **Sizing inconsistency:** I initially claimed that two identical workflow runs produced different story sizes even though the temperature was set to 0. I presented this as evidence of non-deterministic behaviour. When I checked the inputs again, I realised they were different. My conclusion was incorrect, and I retracted it.
- **Cost attribution:** I initially assigned the per-agent costs by looking at the position of nodes in the Langfuse trace tree. When I opened the individual nodes and checked them, I realised I had reversed the mapping. The Scope Estimator, not the Gap Analyser, was the more expensive agent.

Both mistakes came from drawing conclusions based on an initial observation without checking the underlying data.

What made this particularly interesting was that I had been trying to prevent the same kind of assumption-making in PRD Genie itself. I was asking the model not to infer contradictions or invent relationships, but I had made similar assumptions while analysing my own results.

## 5. What I would do differently next time

If I were to build a similar multi-agent workflow again, I would approach a few things differently:

1. **Create the test dataset before writing the prompts.** This would help ensure that I am testing the actual capability of the system rather than checking whether it can reproduce examples already included in its instructions.

2. **Define both failure scenarios from the beginning.** I initially focused on hallucinations and added an evaluator for information loss much later. Having evaluators for both grounding and coverage from the start would have given me a more balanced view of the system.

3. **Use workflow-level checks for important controls.** Instructions alone should not be responsible for enforcing decisions such as whether a workflow can proceed. Approval, validation and stopping conditions should be handled through the workflow itself.

4. **Pay more attention to output formats, templates and examples.** I spent time fixing prompt instructions for issues that were actually being introduced by the output structure or examples. I would review these together rather than treating the prompt as a standalone component.

5. **Improve the Requirement Extractor's input structure.** The current extractor was primarily tuned for meeting transcripts. It does not adequately capture header fields and labelled sections from structured briefs. This is likely a major contributor to the Coverage score of 0.29. I would address this before spending more time refining downstream agents, because information that is missed during extraction cannot be recovered by the later stages.

## Final reflection

Building PRD Genie taught me that getting five agents to perform their individual tasks is only one part of building a reliable multi-agent workflow. The more difficult part is making sure they work together without losing information, introducing assumptions or carrying errors into subsequent stages.

Some of my initial approaches did not work, particularly relying on longer prompt instructions to solve problems that needed structural changes. The evaluation results also showed that I had focused on one type of failure while overlooking another.

The most useful outcome for me was not just getting the workflow to run end to end, but understanding where it could fail, how to identify those failures and what I would change in a future implementation.