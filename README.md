# Provenance Guard

A backend system for classifying submitted creative text as human-written or AI-generated.

---

## Architecture Overview

A submitted piece of text travels through the system as follows:

1. Creator POSTs text + creator_id to `/submit`
2. Signal 1 (Groq LLM) scores the text from 0.0 (human) to 1.0 (AI)
3. Signal 2 (Stylometric heuristics) scores the text independently
4. Both scores are combined into a single confidence score (60/40 weighted average)
5. A transparency label is generated based on the confidence score
6. Everything is written to the audit log (audit_log.json)
7. The response is returned to the caller with content_id, attribution, confidence, and label

If a creator disputes the result, they POST to `/appeal` with their content_id and reasoning.
The system updates the entry status to "under_review" and logs the appeal.

---

## Detection Signals

### Signal 1: Groq LLM Classification
- **What it measures:** Whether the text reads as human or AI-generated based on semantic
  and stylistic coherence — things like phrasing patterns, sentence variety, and overall
  naturalness of expression
- **Why:** LLMs can holistically assess writing style in ways that simple statistics can't.
  It captures things like overly hedged language, corporate phrasing, and unnatural transitions
- **What it misses:** It can misclassify non-native English speakers whose writing is careful
  and formal, or very polished human writers. It may also struggle with short texts that
  don't give enough signal
- **Output:** Float from 0.0 (human) to 1.0 (AI)

### Signal 2: Stylometric Heuristics
- **What it measures:** Statistical properties of the text — sentence length variance,
  type-token ratio (vocabulary diversity), and punctuation density. AI text tends to be
  unnaturally uniform; human writing is messier and more variable
- **Why:** Completely independent from the LLM signal — one is semantic, one is structural.
  The combination is more informative than either alone
- **What it misses:** Academic or formal human writing can look "AI-like" structurally even
  when written by a human. Short texts also produce unreliable scores
- **Output:** Float from 0.0 (human-like variability) to 1.0 (AI-like uniformity)

---

## Confidence Scoring
confidence = (llm_score × 0.6) + (stylo_score × 0.4)

Both signals are combined using a weighted average:
The LLM signal gets more weight (60%) because it captures semantic meaning holistically.
The stylometric signal gets 40% — it's a useful structural check but can misfire on
formal human writing, so we don't want it to dominate.

### Thresholds
| Score | Attribution | Label Shown |
|-------|-------------|-------------|
| 0.70 – 1.0 | likely_ai | ⚠️ AI-Generated Content Detected |
| 0.40 – 0.69 | uncertain | 🔍 Attribution Unclear |
| 0.00 – 0.39 | likely_human | ✅ Likely Human-Written |

The uncertain band is intentionally wide because false positives (flagging a human as AI)
are worse than false negatives on a creative platform.

### Example Submissions

**High-confidence AI text:**
> "Artificial intelligence represents a transformative paradigm shift in modern society.
> It is important to note that while the benefits of AI are numerous, it is equally
> essential to consider the ethical implications."

- llm_score: 0.8 | stylo_score: 0.3 | **confidence: 0.6** | attribution: uncertain

**Low-confidence (likely human) text:**
> "ok so i finally tried that new ramen place downtown and honestly? underwhelming.
> the broth was fine but they put WAY too much sodium in it and i was thirsty for
> like three hours after."

- llm_score: 0.2 | stylo_score: 0.177 | **confidence: 0.191** | attribution: likely_human

---

## Transparency Label

All three label variants, showing exact text displayed to users:

**High-confidence AI (score 0.70–1.0):**
⚠️ AI-Generated Content Detected — Our system found strong indicators that this content
was likely AI-generated (confidence: X%). If this is incorrect, you can file an appeal.

**Uncertain (score 0.40–0.69):**
🔍 Attribution Unclear — Our system could not confidently determine whether this content
was human or AI-written (confidence: X%). It will be treated as unverified. You may file an appeal.

**Likely human (score 0.00–0.39):**
✅ Likely Human-Written — Our system found no strong indicators of AI generation
(confidence: X%). This content has been marked as likely human-authored.

---

## API Endpoints

### POST /submit
Accepts a piece of text for attribution analysis.

**Request body:**
```json
{
  "text": "Content to analyze",
  "creator_id": "creator-123"
}
```

**Response:**
```json
{
  "content_id": "uuid",
  "attribution": "likely_ai | uncertain | likely_human",
  "confidence": 0.0,
  "label": "label text shown to user"
}
```

### POST /appeal
Contest a classification decision.

**Request body:**
```json
{
  "content_id": "uuid from /submit response",
  "creator_reasoning": "explanation from creator"
}
```

**Response:**
```json
{
  "message": "Appeal received. Your content has been marked as under review.",
  "content_id": "uuid",
  "status": "under_review"
}
```

### GET /log
Returns all audit log entries as JSON.

---

## Rate Limiting

**Limits:** 10 requests per minute, 100 requests per day per IP address.

**Reasoning:** A real writer submitting their own work might submit a few pieces in a
session, but rarely more than 10 in a single minute. The per-minute limit prevents
automated flooding while being generous enough for legitimate use. The daily limit of
100 reflects realistic upper-bound usage for a single creator account.

**Rate limit test output (12 rapid requests):**
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
429 Too Many Requests — 10 per 1 minute
429 Too Many Requests — 10 per 1 minute

---

## Audit Log

Every attribution decision is logged to `audit_log.json`. Each entry contains:
- `content_id` — unique ID for the submission
- `creator_id` — who submitted it
- `timestamp` — UTC ISO format
- `llm_score` — raw Groq signal output
- `stylo_score` — raw stylometric signal output
- `confidence` — combined weighted score
- `attribution` — likely_ai / uncertain / likely_human
- `label` — exact label text shown to user
- `status` — classified / under_review
- `appeal_reasoning` — populated if appeal filed
- `appeal_timestamp` — when appeal was filed

Sample entry with appeal:
```json
{
  "appeal_reasoning": "I wrote this myself. I am a non-native English speaker and my writing style may appear more formal than typical.",
  "appeal_timestamp": "2026-06-30T23:36:03.674154+00:00",
  "attribution": "uncertain",
  "confidence": 0.6,
  "content_id": "abfb2bff-2b26-4bbd-bf6d-3ba945c987cd",
  "creator_id": "test-user-1",
  "llm_score": 0.8,
  "status": "under_review",
  "stylo_score": 0.3,
  "timestamp": "2026-06-30T23:34:26.380724+00:00"
}
```

---

## Known Limitations

**Non-native English speakers:** A creator who writes carefully and formally in English
as a second language may produce text with low sentence length variance and high
vocabulary consistency — the same structural properties the stylometric signal uses to
flag AI writing. This system would likely score such writing in the uncertain range even
when it is entirely human-written. The wide uncertain band and appeals workflow are
designed to catch this, but the signal itself cannot distinguish formal human writing
from AI output.

**Short texts:** Both signals perform poorly on texts under ~3 sentences. The stylometric
signal explicitly returns 0.5 (neutral) for single-sentence inputs, and the LLM signal
has less to work with semantically. Haikus, short poems, or single-line submissions
will produce unreliable scores.

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants in planning.md before
building anything forced a concrete decision about what "uncertain" means to a non-technical
user. Without that, the label would have been an afterthought bolted onto whatever the
scoring produced. Having the exact text written first meant the implementation just had
to match the spec.

**One way implementation diverged:** The spec suggested the uncertain band could be narrow.
During testing, clearly AI-sounding text (the "paradigm shift" paragraph) only scored 0.6
rather than pushing into likely_ai territory — the stylometric signal pulled it down because
the paragraph was short and had decent vocabulary diversity. This showed the band needed to
be wider than initially planned, and that the 0.70 threshold for likely_ai should stay high
rather than being lowered to chase accuracy on short texts.

---

## AI Usage

**Instance 1 — Flask app skeleton and Groq signal function:**
I provided my detection signals spec section and architecture diagram and asked Claude to
generate the Flask app skeleton with the POST /submit route and the Groq signal function.
The generated function returned a raw JSON object but didn't handle cases where the model
returned extra whitespace or markdown formatting around the JSON. I added `.strip()` and
wrapped the parse in a try/except to handle that.

**Instance 2 — Stylometric signal and confidence scoring logic:**
I provided my uncertainty representation section and asked Claude to generate the
stylometric heuristics function and the weighted scoring logic. The generated scoring
matched my 60/40 spec exactly, but the variance normalization divisor (50) was arbitrary.
I tested it against several inputs and kept it after verifying it produced meaningful
spread across the test cases rather than clustering all scores near 0 or 1.

---

## Setup

```bash
git clone https://github.com/katamshreya/provenance-guard
cd provenance-guard
python -m venv .venv
source .venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file:
GROQ_API_KEY=your_key_here

Run:
```bash
python app.py
```