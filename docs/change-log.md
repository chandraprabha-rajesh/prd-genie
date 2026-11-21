# PRD Genie — Prompt Change Log

Every prompt change made during the build, what caused it, and whether it worked.
This is the evidence base for Q4 and the reference for explaining the project to
someone else.

Read the **Lessons** section at the end first if you are short on time — it is the
summary of everything below.

---

## AGENT 1 — Requirement Extractor

Stable from early on. Passed T1, T2 and T9 without prompt rework.

| # | Problem | Fix | Result |
|---|---|---|---|
| 1.1 | Read File would not connect to downstream components | Output mode changed from "Files" to "Raw Content"; one file per component | Fixed. Tooling issue, not a prompt issue. |

**Open issue, not yet fixed:** Agent 1 places action items and timelines inside
"Stated Requirements" — e.g. "Export permissions will be confirmed by 25 Nov" and
"Release timeline is Q1" were listed as requirements. Agent 2 filtered them out
correctly, but by its own judgement, not by rule.

---

## AGENT 2 — PRD Generator

| # | Problem | What was tried | Result |
|---|---|---|---|
| 2.1 | Assigned "Must" priority to requirements nobody had prioritised | Three separate rules forbidding invented priority | **All three failed** |
| 2.2 | Same problem | Diagnosis: the PRD template's column header `Priority (Must/Should/Nice)` was reading as an instruction to choose. Rule added high in the list stating header values are permitted values, not a prompt to pick one | **Worked** |
| 2.3 | Invented `Product Name: Report Filtering Feature` and `Document Version: 1.0` | Example showing "Not stated", plus an explicit note that a name derived from the requirements is still an invention | Worked |
| 2.4 | Reproduced the template's instructional text in the output ("For each persona, describe:") | Rule distinguishing template *structure* from template *instructional text* | Worked |
| 2.5 | Generated a full PRD when the extraction said "Stated Requirements: None" and the PM said "go ahead" | Four attempts: a rule that PM feedback can only REMOVE/CORRECT/CLARIFY; rewriting the INPUT section; removing a capitalised "PM FEEDBACK WINS" line; testing with contradictory feedback | **All four failed** |
| 2.6 | Same problem | Refusal moved out of prose and into OUTPUT FORMAT as a STEP 1 / STEP 2 branch. All duplicate refusal text removed — it had appeared four times in three different wordings. Example 3 given its own correct output (the refusal example had drifted under Example 4, which is about a case that *does* have requirements) | **Worked, both with "do not create a PRD" and with "looks fine, go ahead"** |
| 2.7 | Invented `Status: Not started` and `Risk: Medium` in the Dependencies table; dropped the stated "25 Nov" date | Rule 3a: template columns hold information the source gave; absent values are "Not stated". Second paragraph: stated dates, names and numbers must survive | Worked for Risk/Status and the date |
| 2.8 | Put `UNRESOLVED: whether PDF export is needed` into **Out of Scope** | Rule 3b: Out of Scope holds only what someone explicitly excluded; unresolved items go to Open Questions only, and never to both | Worked |
| 2.9 | **Regression caused by 2.7**: rule 3a's "put it in the nearest appropriate field" led the model to fabricate an entire Acceptance Criteria section from requirement text, and declare it in Source Traceability as "Derived from stated requirements" | Rule narrowed: named the fields details may move into, and added an explicit prohibition on deriving acceptance criteria | Worked |
| 2.10 | **Regression caused by 2.7/2.8**: Priority returned to "Must" for all three requirements after the new rules were added — 2.2 had been stable for many runs | Instead of another rule, the PRD template's column header was amended from `Priority (Must/Should/Nice)` to `Priority (Must / Should / Nice / Not stated)` | **Worked.** The template is part of the prompt. |

---

## AGENT 3 — Story Breakdown

| # | Problem | What was tried | Result |
|---|---|---|---|
| 3.1 | Non-functional requirements became user stories ("As a user, I want results to load in under 2 seconds") | Rule 10 forbidding it | **Failed** |
| 3.2 | Same problem | `## Technical Tasks` added to OUTPUT FORMAT, and the rule moved to position 4 | **Worked.** A rule saying "don't put it there" fails until the format says where to put it instead. |
| 3.3 | Produced `Priority: Must Have (stated in PRD — FR-001)` for a requirement the PRD had marked "Not stated" — a false citation | Caused by a two-branch rule: "cite the PRD if it states a priority, otherwise mark suggested". Reverted to a single branch: priority is **always** labelled suggested, with its basis | Worked. The false citation was more dangerous than the original invention, because it signalled verification. |
| 3.4 | Copied requirement text into the "Acceptance criteria" field, which made Agent 5 believe criteria existed | Output format line changed: criteria are copied from the PRD, and if absent the field reads exactly `None in PRD.` Explicit note that a requirement is not an acceptance criterion | Worked, and improved Agent 4 as a side effect — see Lesson 8 |
| 3.5 | Given Agent 2's three-line refusal, invented an entire authentication product: registration, login, password recovery, email/password criteria, security compliance. None of it appeared in any input | STEP 1 input guard added to OUTPUT FORMAT: if the input is not a PRD containing requirements, output two lines and stop | Worked |

---

## AGENT 4 — Gap Analyzer

| # | Problem | What was tried | Result |
|---|---|---|---|
| 4.1 | Fabricated an entire stakeholder debate about multi-select options that never occurred, and raised a "load time" gap for something the source stated explicitly | Rules requiring the source line creating each gap to be quoted; "Unresolved decisions" restricted to contradictions present in the source | Worked |
| 4.2 | No input validation | STEP 1 input guard added | Worked |

**Open issue, not fixed:** the "Unresolved decisions" section reports `None` even when
the output itself tags an item `Type: UNRESOLVED` elsewhere. The section name and the
rule behind it disagree — the rule treats it as "contradictions", the name promises
"unresolved decisions".

---

## AGENT 5 — Scope Estimator

| # | Problem | What was tried | Result |
|---|---|---|---|
| 5.1 | Wrote "acceptance criteria fully specified in the source" for stories whose PRD marked Acceptance Criteria as "Not discussed", and concluded "the sizing is reliable" | Three new rules forbidding the claim | **All three failed, output unchanged** |
| 5.2 | Same problem | Root cause found in the prompt's own **Example 1**, which contained the exact string `acceptance criteria fully specified in the source` on a story nearly identical to the test input — the model was copying it. The SIZING GUIDE also *defined* S as "criteria fully specified", so calling anything S required the claim | Example rewritten on a different story; "criteria fully specified" removed from the S definition; Example 1b added showing the correct wording when criteria are absent. **Worked** |
| 5.3 | No input validation | STEP 1 input guard added | Worked |
| 5.4 | Rule 5 says a story blocked by the gap analysis is UNSIZEABLE, but Agents 4 and 5 run in parallel so Agent 5 never receives the gap analysis | Rule made conditional on the gap analysis actually being present | Resolved in prompt. The underlying disagreement is architectural — see Lesson 7 |

---

## LESSONS

**1. Output structure beats prompt rules.**
Three separate problems resisted rules and were fixed by changing structure:

| Problem | Rules | Structure |
|---|---|---|
| Invented priority | 3 attempts failed | Column header fixed → worked |
| NFRs as user stories | failed | `## Technical Tasks` section added → worked |
| Refusing to generate | 4 attempts failed | STEP 1 branch in OUTPUT FORMAT → worked |

**2. Examples beat rules.**
Agent 5 ignored three explicit prohibitions because one example in the same prompt
demonstrated the forbidden behaviour, on a story that closely resembled the test
input. The model copied what it was shown, not what it was told. Examples that mirror
the test data get copied verbatim rather than generalised.

**3. The template is part of the prompt.**
Changing one column header from `Priority (Must/Should/Nice)` to
`Priority (Must / Should / Nice / Not stated)` fixed a failure that three rules could
not. A list of options with no "none" option reads as a forced choice.

**4. Offering a way to cite a source creates a way to fake one.**
A two-branch priority rule produced `Must Have (stated in PRD — FR-001)` against a
PRD that said "Not stated". Removing the branch removed the failure. A false citation
is worse than an open invention, because it claims verification.

**5. Every added rule dilutes the ones already there.**
Changes 2.7 and 2.8 both worked, and together they broke 2.2 and introduced 2.9 —
two behaviours that had been stable for many runs. Past a certain prompt length,
fixing one thing costs another.

**6. A refusal in one agent moves fabrication downstream, it does not stop it.**
Agent 2's guard worked. Agent 3 then invented an entire authentication product from
the refusal notice, Agent 5 sized those invented stories S/S/M, and Agent 4 assigned
an owner to a gap inside them. In a sequential pipeline a refusal is only as strong as
the weakest agent after it. Every agent needs to validate its own input.

**7. Parallel branches cannot agree with each other.**
Agents 4 and 5 run in parallel. Agent 4 marked three stories BLOCKING while Agent 5
sized them confidently in the same run. Neither is wrong on its own input. This is the
cost of the parallel design, not a bug in either prompt.

**8. Honest output upstream improves agents downstream, without touching them.**
When Agent 3 began writing `Acceptance criteria: None in PRD.` instead of restating
the requirement, Agent 4 started flagging missing criteria as BLOCKING. Agent 4's
prompt was not changed.

**9. Temperature 0 does not produce deterministic documents.**
Three runs of the same input through the same prompts produced three different PRDs:

| Run | PDF export (an unresolved item) | Dependencies |
|---|---|---|
| 1 | in Open Questions | "Not discussed" |
| 2 | in Out of Scope, removed from Open Questions | "Not discussed" |
| 3 | in both | full table, with an invented Risk rating |

The variation was in document content and in which section a decision landed, not in
wording. A string-matching evaluator would have scored all three as passes.

**10. An explicit human instruction can override a system rule.**
Before the structural fix, an extraction containing no requirements plus the PM
feedback "looks fine, go ahead" produced a full PRD, across four prompt revisions.
A rule in a system prompt competes poorly against a direct request in the input.
This is the core argument for the human checkpoint carrying a control, not text.

**11. Review without authority is not a control.**
The human-in-the-loop checkpoint was correctly placed, and the human's corrections
worked. What failed was the veto: a reviewer's "no" delivered as a sentence is a
suggestion. Corrections belong in text; go/no-go belongs in a conditional.
