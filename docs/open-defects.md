# PRD Genie — Open Defects

Found during the T3–T10 baseline test run (20 Nov). None fixed — recorded for the
writeup and for a later pass if time allows.

Ordered by severity.

---

## HIGH — a claim that was not in the source

### D1. Agent 1 fabricates contradictions between unrelated decisions
**Found in:** T6
**Input:** `Engineering wants microservices. Design wants single-page app. PM wants it shipped by March.`
**Output:**
```
Contradictions
Engineering wants microservices, while Design wants a single-page app.
Both were stated. The trade-off was not resolved.
```
**Why it is wrong:** these do not conflict. Microservices is a backend decomposition;
a single-page app is a frontend approach. Most products use both. The output asserts
a trade-off and never names what the trade-off is.

**Likely cause:** Rules 3 and 4 emphasise capturing disagreement, and Example 3 shows
two stated positions in conflict. Two stakeholders wanting different things
pattern-matches that example even when the two things are compatible.

**Candidate fix (untested):** require the conflict to be stated, not inferred —
*"A contradiction exists only where the two statements cannot both be true. State
precisely what cannot hold at the same time. If you cannot name what is mutually
exclusive, it is not a contradiction — record both positions under Stated
Requirements and move on."* This mirrors the constraint that fixed the Grounding
evaluator: a finding requires evidence.

---

### D11. Agent 1 asserts agreement that was never recorded
**Found in:** T8
**Input:** `Admins need bulk user management. End users need a simplified view. Auditors need read-only access with full history.`
**Output:**
```
Decisions vs Open Items
DECIDED: The requirements for admins, end users, and auditors were agreed upon.
```
Nobody agreed anything. The source states three needs. "Were agreed upon" is a claim
about consensus that the source does not make, and Rule 5 exists precisely to keep a
stated need out of DECIDED.

**Same family as D1.** Agent 1 invents *relationships* between statements — sometimes
a conflict (D1), sometimes an agreement (D11). Both are unstated.

**Candidate fix (untested):** *"DECIDED contains only decisions the source records as
made — someone confirming, approving or settling something. A stated need is not a
decision. If no decision is recorded, write 'DECIDED: None.'"*

---

## MEDIUM — structure and duplication

### D2. Agent 3 outputs the same requirement as both a story and a technical task
**Found in:** T6
```
Story 1.1.1  As a developer, I want the product to use microservices...
Technical Tasks
The product must use microservices. (Constrains Story 1.1.1.)
```
A functional requirement became a story *and* a constraint on itself. Technical Tasks
should hold non-functional requirements only.

### D3. Agent 5 reproduces its own output-format instruction text
**Found in:** T6
```
Unsizeable
For each: story, what is missing, which gap or question must be closed first.
Story 1.1.1: "..."
```
The first line is the instruction from Agent 5's OUTPUT FORMAT, not content. Agent 2
had this same defect and it was fixed there with a rule separating template structure
from template instructional text. Agent 5 never received that rule.

### D4. Agent 2 fills empty table sections with placeholder rows
**Found in:** T3, T6
```
7. Dependencies
Dependency | Owner | Status | Risk
Not stated. | Not stated. | Not stated. | Not stated.
```
```
4.2 Non-Functional Requirements
NFR-001 | Not discussed. | Not discussed | Not discussed
NFR-002 | Not discussed. | Not discussed | Not discussed
```
Where a section has no content the section should read `Not discussed.` once. A table
with a row of "Not stated" implies a dependency exists. NFR-002 is a second empty row
for nothing at all.

### D5. Agent 1 omits headings that should always be present
**Found in:** T3
`## Ambiguous Items` did not appear, though "Performance is critical" has no target
and belonged there. The prompt says *"Include every heading. Write 'None.' where
empty."*

---

### D9. Agent 2 leaves Dependencies empty when the requirements name an external system
**Found in:** T7
FR-001 captured `API must integrate with Salesforce REST API v52` — an external
system with a pinned version. Section 7 Dependencies read `Not stated.` A named
external system is a dependency and belongs in both places. Nothing was invented;
something real failed to reach the section built for it. A grounding check cannot
catch this, which is the argument for the Coverage evaluator.

### D10. Agent 1 under-flags ambiguity when a vague phrase sits inside a complete sentence
**Found in:** T8
`Ambiguous Items: None.` and `Missing Information: None.`, although "a simplified
view" and "full history" are undefined. The Gap Analyzer caught all three downstream
and marked them BLOCKING. In T2 the same agent correctly flagged "better reporting",
so the behaviour is inconsistent — a vague term inside a grammatically complete
requirement ("End users need a simplified view") reads as specified; the same term
alone ("we need better reporting") does not.

---

## LOW — cosmetic

### D6. Agent 1 repeats the requirement when it matches the source phrase
**Found in:** T3
```
The dashboard should auto-refresh every 5 seconds. ("The dashboard should auto-refresh every 5 seconds.")
```
Correct per the format (requirement, then supporting phrase in brackets), but when the
source sentence is already a clean requirement the two are identical.
**Candidate fix:** *"If the requirement and the supporting phrase are identical, write
the phrase once, without brackets."*

### D8. Agent 1 writes its own label in place of the source quote
**Found in:** T7
```
API must integrate with Salesforce REST API v52. (Integration requirement)
```
The output format says the bracket holds the supporting phrase from the input.
`(Integration requirement)` is a classification Agent 1 wrote, not a quote. Harmless
here, but the bracket is the traceability mechanism — if it can hold the agent's own
words, it stops being evidence.

### D7. Agent 5 infers technical complexity the source did not describe
**Found in:** T3
> "Single surface with a performance constraint; **one integration with the API**
> mentioned in the source."

The source said "minimize API calls". It did not describe an integration to build.
Agent 5's own rule 4 forbids inferring technical complexity the source did not state.

---

## CLOSED — looked like a defect, is not

### C1. Agent 1 records acceptance criteria as requirements
**Found in:** T4. The source stated two criteria ("PDF must include company logo",
"CSV must preserve formulas") without labelling them as criteria, which is how real
discussions work. Agent 1 recorded them as requirements, and downstream they became
independent user stories.

**Not a prompt defect.** Distinguishing a requirement from its criterion is an
inference the source did not make, and Agent 1 is forbidden from inferring. The
correction belongs at the human checkpoint — and was demonstrated working: a two-line
PM correction reduced four requirements to two, populated the PRD's Acceptance
Criteria section, and stopped the Gap Analyzer asking circular questions.

### C2. Agent 4 always reports "Unresolved decisions: None"
Earlier runs always showed `None`, which looked like the section was broken. T3 showed
it populating correctly when the source contained a genuine unresolved trade-off. The
rule works as designed.
