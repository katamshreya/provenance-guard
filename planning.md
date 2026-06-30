# Provenance Guard — Planning

## Detection Signals

**Signal 1: LLM-based classification (Groq)**
- Measures: whether text reads as AI or human-generated based on semantic/stylistic coherence
- Output: a score from 0.0 (human) to 1.0 (AI)
- Blind spot: may misjudge non-native English speakers or very formal human writing

**Signal 2: Stylometric heuristics**
- Measures: statistical properties — sentence length variance, type-token ratio (vocabulary diversity), punctuation density
- Output: a score from 0.0 (human-like variability) to 1.0 (AI-like uniformity)
- Blind spot: formal essays or academic writing may look "AI-like" structurally even when human-written

## Uncertainty Representation

Confidence score = weighted average of both signals:
- LLM score: 60% weight (more reliable, semantic understanding)
- Stylometric score: 40% weight (structural, can misfire on formal human writing)

Thresholds:
- 0.0 – 0.39 → Likely human-written (low AI confidence)
- 0.40 – 0.69 → Uncertain (cannot confidently classify)
- 0.70 – 1.0  → Likely AI-generated (high AI confidence)

A score of 0.5 means the system genuinely cannot tell — both signals are giving mixed
or moderate readings. This is intentionally a wide uncertain band because false positives
(flagging a human as AI) are worse than false negatives on a creative platform.

## Transparency Label Variants

**High-confidence AI (score 0.70–1.0):**
"⚠️ AI-Generated Content Detected — Our system found strong indicators that this content
was likely AI-generated (confidence: [X]%). If this is incorrect, you can file an appeal."

**Uncertain (score 0.40–0.69):**
"🔍 Attribution Unclear — Our system could not confidently determine whether this content
was human or AI-written. It will be treated as unverified. You may file an appeal."

**High-confidence human (score 0.0–0.39):**
"✅ Likely Human-Written — Our system found no strong indicators of AI generation
(confidence: [X]%). This content has been marked as likely human-authored."

## Appeals Workflow

- Who can appeal: any creator who submitted the content (identified by creator_id)
- What they provide: content_id + a written explanation (creator_reasoning)
- What the system does when appeal is received:
  1. Updates the content's status from "classified" to "under_review"
  2. Logs the appeal alongside the original classification in the audit log
  3. Returns a confirmation message to the creator
- What a human reviewer would see: the original text, both signal scores, the
  confidence score, the label that was shown, and the creator's reasoning

## Edge Cases

1. Non-native English speakers: formal or careful phrasing may score high on
   stylometrics even though it's human-written. The wide uncertain band (0.40–0.69)
   is designed to catch these cases rather than hard-flagging them.

2. Lightly edited AI output: a human who takes AI output and edits it heavily may
   produce text that scores in the uncertain range. The system cannot distinguish
   this from careful human writing — it will show the uncertain label, which is the
   honest answer.

   ## Architecture

### Submission Flow
POST /submit
│
├──► Signal 1: Groq LLM score (0.0–1.0)
│
├──► Signal 2: Stylometric heuristics score (0.0–1.0)
│
├──► Confidence scoring (weighted average → 0.0–1.0)
│
├──► Transparency label (based on score thresholds)
│
├──► Audit log entry (timestamp, scores, label, status)
│
└──► JSON response (content_id, attribution, confidence, label)

### Appeal Flow
POST /appeal
│
├──► Look up content_id in audit log
│
├──► Update status → "under_review"
│
├──► Append appeal_reasoning to log entry
│
└──► JSON response (confirmation message)

## AI Tool Plan

### M3 (Submission endpoint + Signal 1)
- Spec sections to provide: Detection Signals + Architecture diagram
- Ask AI to generate: Flask app skeleton with POST /submit route stub + Groq LLM
  signal function
- Verification: call the Groq function directly with 2-3 test inputs and inspect
  the score before wiring into the endpoint

### M4 (Second signal + Confidence scoring)
- Spec sections to provide: Detection Signals + Uncertainty Representation + Architecture diagram
- Ask AI to generate: stylometric heuristics function + confidence scoring logic
  that combines both signals using 60/40 weighting
- Verification: test with clearly AI text vs clearly human text — scores should
  differ noticeably, and map to different label tiers

### M5 (Production layer)
- Spec sections to provide: Transparency Label Variants + Appeals Workflow + Architecture diagram
- Ask AI to generate: label generation function + POST /appeal endpoint
- Verification: confirm all three label variants are reachable by submitting inputs
  at different confidence levels; test appeal updates status to "under_review" in log