# Flow Verification — the two flows differ by one edge

Both LangFlow exports were compared programmatically rather than by eye, because the
prompts had been copied between flows by hand several times during the build and a
silent drift had already been found once.

## 1. No credentials in the exports

Both files were exported with **"Save with my API keys" unchecked**. Scanned for
`sk-` prefixed strings and for populated `api_key` value fields:

| File | `sk-` matches | Populated API key values |
|---|---|---|
| `PRDGenie_HumanInTheLoop.json` | 0 | 0 |
| `PRDGenie_AutomatedChain.json` | 0 | 0 |

The 45 `api_key` occurrences in each file are provider metadata — field *names*
(`api_key_param: "api_key"`), not values. Safe to publish.

## 2. The five prompts are byte-identical

SHA-256 of each system prompt, truncated:

| Node | Length | HITL | Automated | |
|---|---|---|---|---|
| Requirement extractor | 7,956 | `eb82047b0b` | `eb82047b0b` | identical |
| PRD Generator | 9,813 | `fa5cf0025d` | `fa5cf0025d` | identical |
| Story Writer | 6,756 | `b16fd44021` | `b16fd44021` | identical |
| Gap Analyser | 5,220 | `3f4a4813db` | `3f4a4813db` | identical |
| Scope Estimator | 6,527 | `c1feda9b4c` | `c1feda9b4c` | identical |
| Prompt Template | 80 | `d97b06bb3e` | `d97b06bb3e` | identical |

This matters for the comparison between the two flows. Any difference in their
output is attributable to the human checkpoint, because there is no other
difference in the instructions.

## 3. The flows differ by exactly one edge

**Automated Chain**

```
Read File_Ingestion ──→ Requirement extractor ──→ Prompt Template ──→ PRD Generator
Read File_PRD_template ───────────────────────────↗
                                                     │
                                            Story Writer
                                              ├──→ Gap Analyser
                                              └──→ Scope Estimator
```

**Human in the Loop** — the same graph with one edge removed and one added:

| | Automated | Human in the Loop |
|---|---|---|
| Extractor → Prompt Template | present | **absent** |
| Chat Input → Prompt Template | absent | **present** |

Everything downstream of the Prompt Template is identical in both.

### What this means

In the Human-in-the-Loop flow the extractor's output goes only to a Chat Output —
it is displayed, and it goes nowhere else. The rest of the pipeline is fed by a
Chat Input, into which the reviewer pastes the extraction they have approved or
corrected.

**The two halves of the pipeline are not connected to each other.** The human is
not a node that reviews a message in transit; the human *is* the edge. Nothing can
reach the PRD Generator that a person did not put there.

That is a stronger guarantee than a review step, and it is visible in the graph
rather than resting on a prompt instruction. A prompt that says "wait for approval"
can be ignored by the model. A missing edge cannot.

## 4. Agents 4 and 5 run in parallel — confirmed

`Story Writer → Gap Analyser` and `Story Writer → Scope Estimator` are two separate
edges from the same source. Neither consumes the other's output. The two therefore
read the same stories independently, and can disagree.

## 5. The Prompt Template contents

```
APPROVED REQUIREMENTS:
{extraction}

---

PRD TEMPLATE TO FOLLOW:
{prd_template}
```

Note that `APPROVED REQUIREMENTS` is literally accurate in the Human-in-the-Loop
flow, where `{extraction}` arrives from the reviewer's Chat Input. In the Automated
Chain it arrives straight from the extractor and nothing has been approved. The
wording was carried across when the flow was duplicated and was deliberately left
unchanged, to keep the two prompt sets byte-identical — which was judged the more
valuable property.
