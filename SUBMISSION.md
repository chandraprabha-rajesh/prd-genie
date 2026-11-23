# PRD Genie — Capstone Submission

**Chandraprabha Rajesh**
Repository: https://github.com/chandraprabha-rajesh/prd-genie

---

## Videos

The three videos are hosted on Google Drive — they are too large to include in this
archive.

| | |
|---|---|
| **Presentation** | https://drive.google.com/file/d/1AabJlZj-um5Z8PK3jhjXJ6UjpYUs3IkB/view |
| **Tool walkthrough — automated chain** | https://drive.google.com/file/d/1OfZUlmp24WAd3U7E2PQu9FDn_2vBt2LH/view |
| **Tool walkthrough — human in the loop and Langfuse** | https://drive.google.com/file/d/1iCLStHRL5qrcybEaD6ck0n1Z2k87lMfH/view |

---

## Where to find things

| Looking for | File |
|---|---|
| **Q3 — Build** | `docs/q3-writeup.md` |
| **Q4 — Reflection** | `docs/q4-reflection.md` |
| **Slide deck** | `docs/PRDGenie_Capstone_Deck.pptx` — narration is in the speaker notes |
| Architecture diagram | `docs/architecture.png` |
| Baseline testing — 14 tests, evaluator scores, the human-in-the-loop comparison | `docs/test-results.md` |
| Known defects, with cause and candidate fix for each | `docs/open-defects.md` |
| Proof the two flows differ by exactly one edge | `docs/flow-verification.md` |
| LangFlow exports, both flows | `flows/` — no credentials in either |
| The five agent prompts | `prompts/` — extracted from the flow export, so they cannot have drifted from what ran |
| The amended PRD template | `templates/prd_template.md` |
| Test inputs | `test-inputs/`, with `heldout/` written after the prompts were frozen |
| Langfuse and LangFlow screenshots | `screenshots/`, indexed in its own README |

`README.md` has the full write-up of what the system does and what is wrong with it.

---

## Headline results

| | |
|---|---|
| Grounding — did the output assert what the source did not say? | **0.66** (258 observations) |
| Coverage — did what the source did say fail to reach the output? | **0.29** (518 observations) |
| Overstated Confidence — code check, no model | **0.98** (259 observations) |
| Baseline tests | 13 of 14 pass, plus 3 held-out inputs |
| Cost per full run | $0.004 · 16K tokens · 50.4s |

1 is the good outcome in all three evaluators. Coverage at 0.29 is the main finding:
every rule in all five prompts was written to prevent invention, and omission turned
out to be more than twice as common.
