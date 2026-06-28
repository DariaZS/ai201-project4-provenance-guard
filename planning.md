# Provenance Guard — planning.md

> Written before implementation. Updated before any stretch features.

---

## Architecture Narrative

A piece of text enters the system via POST /submit. It passes through two independent
detection signals — an LLM-based classifier and a stylometric heuristics analyzer.
Their outputs are combined into a single confidence score, which maps to one of three
transparency label variants. The decision is written to the audit log, and the full
result is returned to the caller.

If a creator disputes the classification, they submit POST /appeal with their
content_id and reasoning. The system updates the content's status to "under review"
and logs the appeal alongside the original decision. No automated re-classification
occurs — a human reviewer handles it from there.

---

## Architecture Diagram
```
POST /submit

                    │

                    ▼

┌─────────────────────────────────────────┐

│           Submission Handler            │

│  - validate input (text, creator_id)    │

│  - generate content_id                  │

└────────────────┬────────────────────────┘

                 │

        ┌────────┴────────┐

        ▼                 ▼

┌──────────────┐   ┌──────────────────────┐

│ Signal 1     │   │ Signal 2             │

│ LLM Classifier│   │ Stylometric Heuristics│

│ (Groq)       │   │ (pure Python)        │

│              │   │                      │

│ Output: 0–1  │   │ Output: 0–1          │

│ (prob AI)    │   │ (prob AI)            │

└──────┬───────┘   └──────────┬───────────┘

       │                      │

       └──────────┬───────────┘

                  ▼

    ┌─────────────────────────┐

    │   Confidence Scoring    │

    │  combined = 0.6llm     │

    │           + 0.4stylo   │

    └────────────┬────────────┘

                 │

                 ▼

    ┌─────────────────────────┐

    │   Transparency Label    │

    │  >= 0.75 → likely AI    │

    │  0.40–0.74 → uncertain  │

    │  < 0.40 → likely human  │

    └────────────┬────────────┘

                 │

        ┌────────┴────────┐

        ▼                 ▼

┌──────────────┐   ┌─────────────┐

│  Audit Log   │   │  Response   │

│  (JSON file) │   │  to caller  │

└──────────────┘   └─────────────┘
            
        POST /appeal

              │

              ▼

┌─────────────────────────────────┐

│  Appeal Handler                 │

│  - look up content_id           │

│  - update status → under_review │

│  - log appeal + reasoning       │

│  - return confirmation          │

└─────────────────────────────────┘
```


---

## Detection Signals

### Signal 1 — LLM Classifier (Groq)
**What it measures:** Semantic and stylistic coherence holistically. The LLM reads
the text and assesses whether it exhibits patterns characteristic of AI generation —
uniform tone, generic phrasing, overly structured argumentation, lack of personal
voice.

**Output:** A float 0–1 representing probability the text is AI-generated.

**What it misses:** Very polished human writing (academic papers, professional essays)
may score high. Highly creative or experimental AI output may score low. Also subject
to the LLM's own biases about what "AI writing" looks like.

### Signal 2 — Stylometric Heuristics (pure Python)
**What it measures:** Statistical surface properties that differ between human and AI
writing:
- **Sentence length variance:** AI text tends toward uniform sentence lengths; human
  writing is more irregular.
- **Type-token ratio (TTR):** Vocabulary diversity. AI text reuses vocabulary more
  predictably; human writing varies more.
- **Punctuation density:** Human writing uses more varied punctuation (dashes, 
  ellipses, exclamation marks); AI tends toward periods and commas.

**Output:** A float 0–1 representing probability the text is AI-generated, derived
from normalizing and averaging the three heuristic scores.

**What it misses:** Non-native English speakers often write with uniform structure
that mimics AI patterns. Formal academic human writing scores high on uniformity.
Very short texts don't provide enough signal.

---

## Confidence Scoring and Uncertainty

**Combination formula:**
```
confidence = 0.6 * llm_score + 0.4 * stylo_score
```
LLM gets higher weight (0.6) because it captures semantic meaning, not just surface
statistics. Stylometrics get 0.4 — useful corroboration but more prone to false
positives on formal human writing.

**Threshold mapping:**

| Score range | Label category | Reasoning |
|-------------|---------------|-----------|
| >= 0.75 | Likely AI-generated | Both signals strongly agree |
| 0.40 – 0.74 | Uncertain | Signals diverge or weakly agree |
| < 0.40 | Likely human-written | Both signals lean human |

**Why 0.40, not 0.50?**
A false positive (labeling human work as AI) is worse than a false negative on a
creative platform. We bias toward human — the uncertain band extends further down
than up, and the human threshold is generous.

---

## Transparency Label Variants

### High-confidence AI (score >= 0.75)
```
⚠️ AI-Generated Content Detected

Our system found strong indicators that this content was likely generated by an AI

tool. This does not mean the work has no value — but readers deserve to know.

Confidence: High. If you wrote this yourself, you can file an appeal below.
```

### Uncertain (score 0.40 – 0.74)
```
🔍 Authorship Uncertain

Our system found mixed signals about whether this content was written by a human

or generated by AI. We're not confident enough to make a clear call.

Confidence: Low-to-moderate. This content is being shown as-is. If you're the

author, you can add context or file an appeal below.
```

### High-confidence human (score < 0.40)
```
✅ Likely Human-Written

Our system found no strong indicators of AI generation. This content appears to

have been written by its credited author.

Confidence: High.
```
---

## Appeals Workflow

**Who can appeal:** Any creator with a valid content_id from a prior /submit response.

**What they provide:**
- `content_id` (str): the ID from their submission
- `creator_reasoning` (str): their explanation in plain language

**What the system does:**
1. Looks up the content_id in the audit log
2. Updates status from "classified" to "under_review"
3. Appends appeal_reasoning and appeal_timestamp to the log entry
4. Returns confirmation: `{"status": "under_review", "message": "Appeal received."}`

**What a human reviewer sees:**
All entries in GET /log with `"status": "under_review"` — including the original
classification scores, the label that was shown, and the creator's reasoning.

---

## Anticipated Edge Cases

**Edge case 1 — Non-native English speaker:**
Formal, grammatically careful writing by a non-native speaker may have low sentence
length variance and high structural uniformity — scoring high on stylometrics even
though it's fully human-written. The uncertain band and appeals workflow are the
safety net here.

**Edge case 2 — Lightly edited AI output:**
A creator who generates text with AI and then edits it heavily may produce output
that scores in the uncertain range — neither signal is confident. The system correctly
returns uncertain rather than forcing a binary verdict.

**Edge case 3 — Very short text:**
A haiku or a two-sentence submission gives stylometrics almost nothing to work with.
TTR and sentence variance are unreliable at short lengths. The system should note
this in the label for short inputs.

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1
**Input to AI:** Detection signals section + architecture diagram + requirements
(Flask, Groq, SQLite/JSON log).
**Ask for:** Flask app skeleton with POST /submit stub + Signal 1 LLM function.
**Verify:** Function returns a float 0–1. Route returns JSON with content_id,
attribution, confidence, label. Test with 2 curl inputs before wiring.

### Milestone 4 — Signal 2 + Confidence scoring
**Input to AI:** Detection signals section + uncertainty representation section +
diagram.
**Ask for:** Stylometric heuristics function + scoring combination logic.
**Verify:** Run 4 test inputs (clear AI, clear human, two borderline). Confirm scores
differ meaningfully. Print both signal scores separately to catch miscalibration.

### Milestone 5 — Production layer
**Input to AI:** Transparency label variants + appeals workflow section + diagram.
**Ask for:** Label generation function + POST /appeal endpoint.
**Verify:** All three label variants reachable. Appeal updates status in log.
Rate limit triggers 429 after threshold.