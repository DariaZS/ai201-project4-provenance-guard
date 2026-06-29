# Provenance Guard 🛡️

A backend system for detecting AI-generated content on creative writing platforms. 
Provenance Guard classifies submitted text using two independent signals, scores 
confidence in that classification, surfaces a plain-language transparency label, 
and handles appeals from creators who believe they've been misclassified.

Built for CodePath AI201 Project 4.

---

## Architecture Overview

A piece of text enters via **POST /submit**. It passes through two independent 
detection signals — an LLM classifier and a stylometric heuristics analyzer. Their 
outputs are combined into a single confidence score, which maps to one of three 
transparency label variants. The full decision is written to a structured audit log 
and returned to the caller.

If a creator disputes the classification, they submit **POST /appeal** with their 
`content_id` and reasoning. The system updates the content's status to "under review" 
and logs the appeal alongside the original decision. No automated re-classification 
occurs — a human reviewer handles it from there.

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

│LLM Classifier│   │ Stylometric Heuristics│

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

### Signal 1 — LLM Classifier (Groq llama-3.3-70b-versatile)
**What it measures:** Semantic and stylistic coherence holistically. The model reads 
the full text and assesses whether it exhibits patterns characteristic of AI 
generation — uniform tone, generic phrasing, overly structured argumentation, 
suspiciously perfect grammar, lack of personal voice.

**Why this signal:** An LLM understands meaning, not just surface statistics. It can 
catch AI writing that varies sentence length deliberately to avoid detection — 
something pure stylometrics would miss.

**Output:** Float 0–1 (probability of AI authorship) + one-sentence reasoning string.

**What it misses:** Very polished human writing (academic papers, professional 
essays) may score high. Highly creative or experimental AI output may score low. 
The model also has its own biases about what "AI writing" looks like, which may 
shift as LLM writing styles evolve.

---

### Signal 2 — Stylometric Heuristics (pure Python)
**What it measures:** Three statistical surface properties:
- **Sentence length variance:** AI text tends toward uniform sentence lengths; 
  human writing is more irregular.
- **Type-token ratio (TTR):** Vocabulary diversity. AI reuses vocabulary more 
  predictably; human writing varies more.
- **Punctuation diversity:** Human writing uses more varied punctuation (dashes, 
  ellipses, exclamation marks); AI tends toward periods and commas.

**Why this signal:** Completely independent of the LLM — it measures structure, not 
meaning. When both signals agree, confidence is higher. When they disagree, the 
uncertain band absorbs the ambiguity.

**Output:** Float 0–1 derived from normalizing and averaging the three heuristic 
scores.

**What it misses:** Non-native English speakers often write with uniform structure 
that mimics AI patterns. Formal academic human writing scores high on uniformity. 
Very short texts (under ~10 words) don't provide enough signal.

---

## Confidence Scoring

**Combination formula:**
```
confidence = 0.6 × llm_score + 0.4 × stylo_score
```
LLM gets higher weight (0.6) because it captures semantic meaning, not just surface 
statistics. Stylometrics get 0.4 — useful corroboration but more prone to false 
positives on formal human writing.

**Threshold mapping:**

| Score range | Attribution | Reasoning |
|-------------|------------|-----------|
| >= 0.75 | `likely_ai` | Both signals strongly agree |
| 0.40 – 0.74 | `uncertain` | Signals diverge or weakly agree |
| < 0.40 | `likely_human` | Both signals lean human |

**Why 0.40, not 0.50?** A false positive (labeling human work as AI) is worse than 
a false negative on a creative platform. The uncertain band extends further down 
than up, biasing toward human when in doubt.

### Example Submissions

**High-confidence AI (confidence: 0.816):**
```
Input: "Artificial intelligence represents a transformative paradigm shift in

modern society. It is important to note that while the benefits of AI are numerous,

it is equally essential to consider the ethical implications..."
llm_score: 0.92

stylo_score: 0.659

confidence: 0.816

attribution: likely_ai
```

**High-confidence human (confidence: 0.237):**
```
Input: "ok so i finally tried that new ramen place downtown and honestly?

underwhelming. the broth was fine but they put WAY too much sodium in it and

i was thirsty for like three hours after..."
llm_score: 0.21

stylo_score: 0.278

confidence: 0.237

attribution: likely_human
```
---

## Transparency Labels

All three variants written out exactly as returned by the API:

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

have been written by its credited author. Confidence: High.
```
---

## Rate Limiting

**Limits:** 10 requests per minute, 100 requests per day per IP address.

**Reasoning:** A legitimate writer submitting their own work might submit 2–3 pieces 
in a session, occasionally burst to 5–6 if revising and resubmitting. 10/minute 
accommodates normal usage with headroom. 100/day prevents a script from flooding 
the system overnight while still allowing a prolific writer's full day of work. 
A 429 response is returned when the limit is exceeded.

**Rate limit test output (12 rapid requests):**
```
200

200

200

200

200

200

200

200

200

200

429

429
```
---

## Audit Log

Every submission and appeal is logged to `audit_log.json`. Sample entries:

```json
[
  {
    "content_id": "cf8f90d7-27b0-4b11-9160-e86fde5367af",
    "creator_id": "u_demo",
    "timestamp": "2026-06-28T17:58:55.711771+00:00",
    "attribution": "likely_ai",
    "confidence": 0.816,
    "llm_score": 0.92,
    "llm_reasoning": "The text exhibits a uniform tone, generic phrasing, and overly structured argumentation.",
    "stylo_score": 0.659,
    "status": "classified",
    "appeal_reasoning": null,
    "appeal_timestamp": null
  },
  {
    "content_id": "6395b40b-6d81-4e64-8f74-97940a754402",
    "creator_id": "test-user-2",
    "timestamp": "2026-06-28T18:03:49.818148+00:00",
    "attribution": "likely_human",
    "confidence": 0.237,
    "llm_score": 0.21,
    "llm_reasoning": "The text features informal language, personal opinions, and casual tone.",
    "stylo_score": 0.278,
    "status": "under_review",
    "appeal_reasoning": "I wrote this myself from personal experience visiting the restaurant last week.",
    "appeal_timestamp": "2026-06-28T18:04:50.455402+00:00"
  }
]
```

---

## Appeals Workflow

Creators submit **POST /appeal** with their `content_id` and `creator_reasoning`. 
The system updates the entry's status to `under_review`, logs the appeal reasoning 
and timestamp, and returns confirmation. A human reviewer can then query **GET /log** 
to see all entries with `"status": "under_review"` alongside the original 
classification scores and the creator's explanation.

---

## Known Limitations

**Non-native English speakers will be disproportionately flagged.** Careful, 
grammatically precise writing by non-native speakers tends to have low sentence 
length variance and high structural uniformity — exactly what the stylometric signal 
scores as AI-like. A non-native speaker writing a formal essay could score 0.6+ on 
stylometrics even with completely original human writing. The uncertain band and 
appeals workflow are the safety net, but the system should surface this caveat in 
the label for borderline cases.

**Very short texts are unreliable.** The stylometric signal needs at least 2–3 
sentences to compute meaningful variance. A haiku or two-sentence submission gives 
almost no structural signal, defaulting to 0.5 — which pushes the combined score 
into uncertain regardless of what the LLM says.

---

## Spec Reflection

**One way planning.md helped:** The architecture diagram made the conditional flow 
concrete before writing a single line of code. When prompting Claude with the diagram 
for the Flask skeleton, the generated route structure matched the spec exactly on the 
first attempt — the diagram communicated the two-flow design (submission vs. appeal) 
more precisely than a text description alone.

**One divergence from the spec:** The planning.md described the stylometric signal 
as combining sentence length variance, TTR, and punctuation density with equal 
weights. During implementation, testing on clearly human vs. clearly AI text showed 
that punctuation diversity was the weakest signal — casual human writers and AI 
writers can both use minimal punctuation. The weights remain equal in the current 
implementation, but a production version would reduce punctuation density's 
contribution after more systematic testing.

---

## AI Usage

### Instance 1 — Flask skeleton + Signal 1 (Milestone 3)
**Input to Claude:** Detection signals section + architecture diagram from 
planning.md + requirements (Flask, Groq, JSON audit log).  
**Output:** Complete `app.py` with POST /submit, GET /log, the LLM signal function, 
and audit log helpers.  
**What I revised:** The LLM prompt initially asked for a binary "human/AI" 
classification. I changed it to request a float score 0–1 with a one-sentence 
reasoning field — this gives the confidence scoring layer something to work with 
and surfaces the LLM's reasoning in the audit log.

### Instance 2 — Stylometric signal + confidence scoring (Milestone 4)
**Input to Claude:** Detection signals section + uncertainty representation section 
+ diagram.  
**Output:** Stylometric heuristics function computing sentence length variance, TTR, 
and punctuation diversity, plus the 0.6/0.4 weighted combination formula.  
**What I revised:** The original TTR normalization assumed TTR < 0.4 was AI-like. 
Testing showed most short texts naturally have high TTR regardless of authorship — 
I adjusted the normalization range to 0.5–0.8 to better separate the signal from 
noise on realistic inputs.

---

## Setup

```bash
git clone https://github.com/YOUR_USERNAME/ai201-project4-provenance-guard
cd ai201-project4-provenance-guard
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "GROQ_API_KEY=your_key_here" > .env
python app.py
```

API runs at `http://127.0.0.1:5001`.

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/submit` | Submit text for classification |
| POST | `/appeal` | Appeal a classification |
| GET | `/log` | View audit log (last 20 entries) |