# ai201-project4-provenance-guard
A backend API system that classifies submitted creative text as human-written or AI-generated, returns a transparency label, and handles creator appeals. Built for CodePath AI201 Project 4.
# Provenance Guard

---

## Architecture Overview

A submitted piece of text takes the following path through the system:

1. **POST /submit** receives the text and creator ID
2. **Signal 1 (LLM classification)** — sends the text to Groq (llama-3.3-70b-versatile) and returns a score from 0.0 (human) to 1.0 (AI)
3. **Signal 2 (Stylometric heuristics)** — computes sentence length variance, type-token ratio, and punctuation density in pure Python, returning a normalized score from 0.0 to 1.0
4. **Confidence scoring** — combines both signals with a weighted formula: `(0.6 × LLM score) + (0.4 × stylometric score)`
5. **Label generation** — maps the confidence score to one of three transparency label variants
6. **Audit log** — writes a structured JSON entry capturing all scores, signals, and metadata
7. **Response** — returns content_id, attribution, confidence score, and label text to the caller

Appeal path: **POST /appeal** accepts a content_id and creator reasoning, updates that entry's status to `under_review`, appends the appeal details to the audit log entry, and returns confirmation.

---

## Detection Signals

### Signal 1: LLM-based classification (Groq)
Sends the submitted text to `llama-3.3-70b-versatile` with a prompt asking it to score the text from 0.0 (clearly human) to 1.0 (clearly AI) and explain its reasoning in one sentence.

**What it captures:** Semantic and stylistic coherence holistically — generic phrasing, symmetrical structure, hedging language, and the overall "feel" that trained models tend to produce.

**What it misses:** AI text that has been lightly edited by a human, or unusually formal human writing (academic papers, legal documents). It's also a model's judgment, not ground truth — it can be confidently wrong.

### Signal 2: Stylometric heuristics (pure Python)
Computes three statistical properties of the text and combines them into a single score:
- **Sentence length variance** — AI text tends toward more uniform sentence lengths; high variance suggests human writing
- **Type-token ratio** — measures vocabulary diversity; AI text often reuses words more than humans do at the same length
- **Punctuation density** — counts informal punctuation (em dashes, ellipses) which humans use more freely than AI

**What it captures:** Structural and statistical regularity independent of meaning — a completely different axis from the LLM signal.

**What it misses:** Short texts (fewer than 2 sentences or 10 words) don't provide enough data for meaningful variance calculation — the signal returns a neutral 0.5 in those cases. Also, poetry with intentional repetition or writers with very uniform style can score as AI-like on structure alone.

**Why these two:** They fail in different ways. The LLM signal is semantic; the stylometric signal is structural. When they agree, confidence is high. When they disagree, that disagreement itself reflects genuine uncertainty.

---

## Confidence Scoring

**Formula:** `confidence = (0.6 × llm_score) + (0.4 × stylometric_score)`

The LLM signal is weighted higher (0.6) because it captures more context. Stylometrics (0.4) pulls the score back when the LLM is overconfident on ambiguous text.

**Thresholds:**
- `0.00 – 0.35` → Likely human-written
- `0.36 – 0.64` → Uncertain
- `0.65 – 1.00` → Likely AI-generated

The uncertain band is deliberately wide because a false positive (calling a human's work AI-generated) is worse than a false negative on a creative platform.

**Example submissions with noticeably different scores:**

High-confidence AI (score: 0.7384):
> "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment."
- LLM score: 0.92 | Stylometric score: 0.4659 | Combined: **0.7384** → `likely_ai`

Lower-confidence human (score: 0.3497):
> "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after."
- LLM score: 0.40 | Stylometric score: 0.2743 | Combined: **0.3497** → `likely_human`

---

## Transparency Label

Three label variants based on confidence score:

| Band | Attribution | Label Text |
|---|---|---|
| 0.65 – 1.00 | `likely_ai` | "This content shows strong signals of AI generation. Our system detected patterns consistent with AI-generated text with high confidence." |
| 0.36 – 0.64 | `uncertain` | "We can't confidently determine whether this content is AI-generated or human-written. This label reflects genuine uncertainty in our detection signals — it is not an accusation." |
| 0.00 – 0.35 | `likely_human` | "This content shows strong signals of human authorship. Our system detected patterns consistent with human-written text with high confidence." |

---

## Rate Limiting

**Limits:** 10 requests per minute, 100 requests per day (applied to POST /submit only)

**Reasoning:**
- A real creator submitting their own work would rarely need more than a few submissions per minute — 10/minute is generous for legitimate use
- 100/day prevents scripted flooding while accommodating heavy legitimate users
- The per-minute limit is the primary abuse guard; the per-day limit catches slow automated scripts that stay under the per-minute ceiling

**Observed behavior:** 10 consecutive requests return `200`, requests 11+ return `429 Too Many Requests`.

---

## Audit Log

Every attribution decision is written to `audit_log.json`. Each entry captures:
- `content_id` — unique UUID for the submission
- `creator_id` — provided by the caller
- `timestamp` — UTC ISO 8601
- `attribution` — `likely_ai`, `uncertain`, or `likely_human`
- `confidence` — combined weighted score
- `llm_score` — raw score from Signal 1
- `llm_rationale` — one-sentence explanation from the LLM
- `stylometric_score` — raw score from Signal 2
- `status` — `classified` or `under_review`
- `appeal` — null, or object containing `timestamp` and `creator_reasoning`

Sample entries visible via `GET /log`.

---

## Appeals Workflow

Creators can contest a classification via `POST /appeal` with their `content_id` and a free-text `creator_reasoning` field.

On receipt the system:
1. Locates the original entry by `content_id`
2. Updates `status` from `classified` → `under_review`
3. Appends the appeal reasoning and timestamp to the log entry
4. Returns confirmation to the caller

No automated re-classification is triggered. A human reviewer would see the original text, both signal scores, the combined confidence, and the creator's reasoning when reviewing the appeal queue via `GET /log`.

---

## Known Limitations

**Non-native English speakers with formal writing styles** are the most likely false positive case. The stylometric signal flags uniform sentence length and low punctuation density as AI-like — properties that are common in second-language writers who write carefully and formally. The LLM signal can also misread careful, structured human prose as AI-generated. The system partially mitigates this with the wide uncertain band (0.36–0.64) and the appeals workflow, but a confident misclassification is still possible for this group.

**Short-form content** (under 2 sentences) gives the stylometric signal no useful data, forcing it to return a neutral 0.5, which reduces the combined score's meaningfulness for haikus, one-liners, or very short poems.

---

## Spec Reflection

**Where the spec helped:** Writing out the three label variants and the confidence thresholds in planning.md before touching any code meant the label generation function had a clear contract to implement against. There was no ambiguity about what text to show at what score range.

**Where implementation diverged:** The planning doc assumed the LLM would reliably return valid JSON. In practice, Groq's model sometimes returned unquoted string values in the rationale field, requiring a regex-based fallback parser instead of a clean `json.loads()` call. The final implementation extracts the score and rationale with separate regex patterns rather than parsing the whole response as JSON.

---

## AI Usage

**Instance 1 — Flask app skeleton and signal functions:** I outlined the two detection signals and the overall submission flow, then used an AI tool to generate a starting point for the Flask skeleton and signal functions. The output needed significant debugging — the Groq function assumed the model would always return clean JSON, which it didn't. I added a debug print to inspect the raw response, identified that the rationale field was coming back unquoted, and rewrote the parsing logic myself using regex to extract the score and rationale independently rather than relying on json.loads.

**Instance 2 — Appeals endpoint and label generation:** I drafted the appeals workflow and label variants in planning.md first, then used an AI tool to generate a rough implementation to react to. I reviewed the appeal endpoint logic, tested it manually end-to-end by submitting an appeal and checking GET /log, and confirmed the status update and reasoning storage worked correctly before accepting it.
