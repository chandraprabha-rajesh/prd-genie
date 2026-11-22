# PRD Genie — Q3: Build

**Repository:** https://github.com/chandraprabha-rajesh/prd-genie
Contains both LangFlow exports, the five agent prompts, the PRD template, the test inputs and the Langfuse screenshots.

## 1. Design and Architecture

The idea behind PRD Genie was to reduce the manual effort involved in converting product discussions into actionable delivery documents. A BA or PM typically has to go through meeting transcripts, emails, briefs and chat discussions to prepare a PRD, break it down into user stories, identify gaps and estimate scope.

During this process, two things can happen: important requirements get missed, or assumptions make their way into the document even though they were never discussed.

I wanted to build a solution that could automate this process while maintaining traceability to the original discussion. My primary design principle was that **the system should capture what was actually discussed, rather than fill in what it assumes should have been discussed.**

### Architecture and orchestration

I evaluated four orchestration patterns: Router, Orchestrator, Hierarchical and Sequential.

A Router is useful when the input type varies and the path has to be chosen; my input is always a discussion document. An Orchestrator is useful when the next step depends on what the previous step found; here the order never changes. A Hierarchical pattern suits work that splits into parts and is merged back together; nothing in this workflow decomposes that way.

I chose a combination of Sequential and Parallel execution based on the dependencies between the agents.

The main workflow follows a sequential approach because each stage needs the output of the previous stage:

1. **Requirement Extractor:** Extracts explicitly stated requirements, decisions and constraints.
2. **PRD Generator:** Converts the extracted information into a structured PRD.
3. **Story Breakdown:** Converts the PRD into epics, features and user stories.

After Story Breakdown, the workflow branches into two parallel agents:

4. **Gap Analyser:** Identifies unanswered questions and missing information before development.
5. **Scope Estimator:** Assigns relative sizes from S to XL.

I chose parallel execution for the last two agents because both consume the same story breakdown and work independently. The Gap Analyser does not need the sizing results, and the Scope Estimator does not need the gap analysis. Running them sequentially would only increase execution time without adding any meaningful dependency.

This gave me a practical combination: sequential execution where there is a dependency and parallel execution where the tasks are independent.

### Tool selection and rationale

I selected **LangFlow over n8n** for building the workflow. n8n is particularly useful when a solution needs integrations with external systems, APIs, scheduled jobs or business applications. PRD Genie did not initially require those integrations. Most of my development effort involved designing and refining prompts and output structures across multiple agents. LangFlow made it easier to manage individual agents, inspect their configurations and visualise the overall workflow.

For observability, I integrated **Langfuse** with LangFlow using environment variables. It captures information at individual agent level, including resolved prompts, actual inputs, token usage, execution time and cost. This was important for identifying where an issue originated instead of trying to diagnose everything from the final output.

I used **gpt-4o-mini with temperature set to 0**. The primary tasks involve requirement extraction, classification and formatting under defined constraints, rather than open-ended reasoning.

## 2. Core Capabilities and Implementation

One of the main challenges was preventing the model from filling gaps simply because the PRD template contained empty sections. I introduced explicit handling for values such as "Not stated", "Not discussed" and "UNKNOWN". These are valid outputs, not failures.

Each agent has a clearly defined responsibility:

- The Requirement Extractor captures only explicitly stated information, records contradictions without resolving them and does not assign owners unless they are mentioned.
- The PRD Generator acts as a formatter rather than an author. Every requirement must be traceable to the extraction, and missing sections remain marked as "Not discussed".
- The Story Breakdown creates stories only from the PRD. Non-functional requirements are separated into technical tasks rather than incorrectly converted into user stories.
- The Gap Analyser identifies missing information and includes the relevant source reference for each gap.
- The Scope Estimator provides relative sizing only. Stories without sufficient information are marked UNSIZEABLE rather than receiving an unsupported estimate.

### Implementation challenges and learnings

One important issue surfaced when I tested the agents together. When the Requirement Extractor received an empty input, it correctly returned a message indicating that there was nothing to process. However, the next agent interpreted that message as valid input and generated an entirely fictional authentication feature, including registration, login and password recovery. The remaining agents then estimated and analysed these invented stories.

This showed me that getting individual agents to behave correctly does not automatically mean the complete pipeline will behave correctly.

I introduced input validation across agents 2–5 so that each agent checks whether it has meaningful input before proceeding.

Another challenge was maintaining the prompts as the workflow grew. Initially, I could make small changes to an individual prompt and test the output. However, as more scenarios were tested, the instructions became longer, with additional rules and examples being added to address each issue. Maintaining these large prompts and making changes without affecting previously working behaviour became increasingly difficult.

I also realised that adding more instructions was not always the best solution. Some issues were better addressed by changing the output structure itself. For example, I moved the NFR heading above stated requirements, added "Not stated" as a priority option and introduced a separate Technical Tasks section. These changes helped prevent the model from making assumptions based on the template.

In each of these cases I had already tried solving the problem with instructions. The empty extraction issue survived four successive prompt revisions, and was resolved on the first attempt once the refusal was moved into the output format as a STEP 1 branch rather than stated as a rule.

This was an important learning: prompt design is not just about adding more instructions. The structure of the prompt and expected output plays an equally important role. Going forward, keeping instructions modular and separating agent responsibilities would make the workflow easier to maintain.

## 3. Human in the Loop — Intended Production Workflow

Although I built an automated version of the pipeline, I consider the Human in the Loop approach the intended production workflow rather than simply an additional feature.

The reason is straightforward: an incorrect requirement extraction can affect every subsequent stage. If an assumption enters the PRD, it can become a user story, generate unnecessary gap questions and even receive a scope estimate. Reviewing the extraction before it moves downstream provides an opportunity to correct these issues at an early stage.

I therefore built a human review checkpoint between the Requirement Extractor and PRD Generator. The PM reviews the extracted requirements, makes corrections and provides the reviewed output to continue the workflow.

In this flow the pipeline is physically discontinuous. The Requirement Extractor's output ends at a Chat Output, and the second half of the workflow begins at a Chat Input. There is no connection between them. A prompt instructing the model to wait for approval can be ignored; a missing connection cannot.

The PM can remove an incorrectly captured requirement, correct an extraction error or clarify an ambiguous statement. However, the checkpoint does not allow entirely new requirements to be introduced, as that would break traceability to the original discussion.

I retained the Automated Chain as a baseline for testing and comparing the impact of human review. Both workflows use identical prompts; the difference is the human review checkpoint. I verified this by hashing each system prompt from both flow exports — all five match, and the two workflows differ by exactly one connection.

**Example from test T4:**

The source contained two acceptance criteria that were not explicitly labelled as such. The automated workflow treated them as separate functional requirements. During human review, I corrected the extraction to clarify that the company logo and CSV formula preservation were acceptance criteria rather than independent functional requirements.

| Measure | Automated Chain | Human in the Loop |
|---|---:|---:|
| Functional requirements | 4 | 2 |
| Acceptance criteria | Not discussed | Both captured |
| Gap questions | 8 | 4 |
| Unanswerable questions | 2 | 0 |
| Gap Analyser tokens | 9.4K | 7.0K |
| Execution time | 34.8 sec | 19.7 sec |

The correction reduced token usage by 26% and execution time by 43%. More importantly, it improved the quality of the questions generated downstream.

For example, the automated workflow asked, "What are the acceptance criteria for including the company logo in the PDF export?" The corrected workflow instead asked what additional details were required for the PDF export beyond the company logo.

This demonstrated that human review does not necessarily add processing overhead. Correcting an issue early can actually reduce unnecessary downstream processing.

I also identified a limitation: approving an empty extraction with a simple "looks fine, go ahead" could still allow the workflow to generate a PRD. Prompt changes alone did not reliably prevent this. The underlying issue is that a correction and a decision are different things. A correction belongs in text, because the model needs to read and apply it. A go or no-go decision written as a sentence becomes one more instruction competing with the rest of the prompt. It belongs in flow logic, not in text.

## 4. Observability and Evaluation

I connected both workflows to Langfuse and used its traces to investigate prompt behaviour, identify where information was lost or invented, and compare individual agent performance.

I created three evaluators to measure different aspects of output quality:

| Evaluation | Type | Observations | Average score |
|---|---|---:|---:|
| Grounding – checks for unsupported information | LLM judge | 258 | 0.66 |
| Coverage – checks whether source information is retained | LLM judge | 518 | 0.29 |
| Overstated Confidence – checks unsupported claims of completeness | Code-based | 259 | 0.98 |

A score of 1 represents the desired result.

I also added a safeguard requiring the LLM evaluators to quote the specific output line before assigning a failing score. This helped avoid unsupported judgements from the evaluator itself. The Overstated Confidence evaluator uses a code-based check, avoiding an additional model call.

The most important finding was the low Coverage score of 0.29. I had initially focused heavily on preventing hallucinations, but the evaluation showed that losing information from the source was an even bigger problem. The extractor was primarily designed around meeting transcripts and did not have an appropriate output structure for all the header fields and labelled sections found in structured briefs. This is an area that needs further improvement.

## 5. Baseline Dataset Testing

I tested the workflow against 14 baseline inputs, with pass/fail criteria defined before execution. Thirteen passed and one failed. I then added three further test cases after freezing the prompts to check whether the workflow could handle scenarios beyond those used during prompt development. All three passed.

The tests identified grounding issues that were not obvious from simply reviewing the generated documents. In particular, the model sometimes created relationships between statements that the source had never connected.

This reinforced the importance of testing against independent scenarios rather than relying only on examples already included in prompts.

## 6. Cost Analysis and Evaluation Strategy

The full five-agent workflow costs approximately **$0.004 per run**, takes 50.4 seconds and uses around 16K tokens. A guarded refusal costs approximately $0.001, takes 14.5 seconds and uses 7K tokens.

The cost is not evenly distributed across the agents:

| Agent | Tokens | Time | Cost |
|---|---:|---:|---:|
| 1. Requirement Extractor | 3.0K | 10.5 sec | $0.00080 |
| 2. PRD Generator | 4.1K | 10.5 sec | $0.00080 |
| 3. Story Breakdown | 3.1K | 9.9 sec | $0.00082 |
| 4. Gap Analyser | 2.5K | 8.6 sec | $0.00042 |
| 5. Scope Estimator | 3.5K | 15.4 sec | $0.00106 |

The Scope Estimator is currently the most expensive agent and also the slowest. Agents 4 and 5 run in parallel, so the elapsed time for that branch is 15.4 seconds rather than the sum of the two.

I also found that output tokens cost approximately four times as much as input tokens. Around 53% of input tokens came from prompt caching, which helps reduce the cost of repeatedly sending the same system prompts.

Based on these results, I retained gpt-4o-mini rather than moving to a larger model. The issues identified during testing were largely related to prompt design, output structure and information handling, rather than the model's reasoning capability.

**Overall, Q3 helped me understand that building a multi-agent workflow is not just about assigning different tasks to different agents. The real challenge is managing dependencies, maintaining consistency across agents, ensuring information is not lost between stages and validating the final output. PRD Genie demonstrates the complete workflow, with human review as an important control before requirements move into downstream delivery activities. The evaluation results also give me a clear direction for further improvement, particularly around coverage and prompt maintainability.**
