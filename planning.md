# Provenance Guard — Backend Flow

## Stack
| Component | Tool |
|---|---|
| API framework | Flask |
| Detection signal 1 | Groq — `qwen/qwen3.8-27b` |
| Detection signal 2 | Stylometric heuristics (pure Python) |
| Rate limiting | Flask-Limiter |
| Audit log | Structured JSON file (`log.json`) |

## Score convention
All scores are on one scale: **0.0 = certainly human-written, 1.0 = certainly AI-generated.**

| Final score | Verdict |
|---|---|
| < 0.35 | `likely_human` |
| 0.35 – 0.65 | `uncertain` |
| > 0.65 | `likely_ai` |

---

## Architecture

```
                  POST /submit   (10/min per IP, else 429)
                  { "creator_id": "...", "text": "..." }
                                 |
                                 v
                 +-------------------------------+
                 |       validate request        |
                 +-------------------------------+
                   |  missing creator_id/text -> 400 Bad Request
                   |  valid -> server generates content_id (uuid4); creators may submit many times
                   v
                 +-------------------------------+
                 |  text_handler(text)           |
                 |  -> list of sentences         |
                 +-------------------------------+
                                 |
                                 v
                 +-------------------------------+
                 |  length_filter(sentences)     |
                 +-------------------------------+
                   |                           |
            < 4 sentences               >= 4 sentences
                   |                           |
                   v                 +---------+---------+
      +-------------------------+    |                   |
      | get_response(           |    v                   v
      |   reason="length")      |  +--------------+  +---------------------+
      | -> "uncertain",         |  | SIGNAL 1     |  | SIGNAL 2            |
      |    score: null          |  | llm_score()  |  | stylometric_score() |
      +-------------------------+  | float / None |  | float               |
                   |               +--------------+  +---------------------+
                   |                    |                   |
                   |                    +---------+---------+
                   |                              v
                   |               +-------------------------------+
                   |               |  combine_scores(llm, stylo)   |
                   |               |  final = 0.7*llm + 0.3*stylo  |
                   |               |  llm None -> final = stylo,   |
                   |               |    "llm_unavailable": true    |
                   |               +-------------------------------+
                   |                              |
                   |                              v
                   |               +-------------------------------+
                   |               |  get_response(reason="score", |
                   |               |    score=final)               |
                   |               |  -> verdict via threshold     |
                   |               +-------------------------------+
                   |                              |
                   +--------------+---------------+
                                  v
                 +-------------------------------+
                 |  add_to_log(entry)            |
                 |  no file -> create []         |
                 |  load -> append -> write back |
                 |  NEVER overwrite/delete       |
                 +-------------------------------+
                                  |
                                  v
         200 OK { content_id, creator_id, attribution, confidence, label }


                  POST /appeal   (5/min per IP, else 429)
                  { "content_id": "...", "creator_reasoning": "..." }
                                 |
                                 v
                 +-------------------------------+
                 |       validate request        |
                 +-------------------------------+
                   |  missing content_id/creator_reasoning -> 400 Bad Request
                   v
                 +-----------------------------------+
                 |  find_post(content_id) (log.json) |
                 +-----------------------------------+
                   |               |                  |
               not found    status=="under_review"  status=="classified"
                   |               |                  |
                   v               v                  v
           404 Not Found    409 Conflict     +------------------------------+
           "No post with    "An appeal for   | update that entry's appeal:  |
            that ID         this post is     |  status: "under_review"      |
            exists."        already under    |  note, filed_at              |
                            review."         | load -> modify -> write back |
                                             | (nothing else changes)       |
                                             +------------------------------+
                                                              |
                                                              v
                                     200 OK "Your appeal has been submitted
                                             and is now under review."
```

**Log entry shape:**
```json
{
  "content_id": "3f2a9c1e-8b7d-4e6a-9f10-2c5d8e7b4a61",
  "creator_id": "test-user-1",
  "timestamp": "2026-10-27T14:03:00.123Z",
  "text": "Full text of the post...",
  "attribution": "likely_ai",
  "confidence": 0.78,
  "llm_score": 0.81,
  "stylo_score": 0.71,
  "combined_score": 0.78,
  "llm_unavailable": false,
  "status": "classified"
}
```

**After an appeal**, the same entry keeps every original classification field and gains:
```json
{
  "status": "under_review",
  "appeal_reasoning": "I wrote this myself from personal experience.",
  "appeal_filed_at": "2026-10-27T15:10:00.456Z"
}
```

---

## Transparency labels
`generate_label(score)` returns the `label` text shown with every result. The score is shown as a percentage "AI-likelihood score".

| Score | Label |
|---|---|
| > 0.65 | **Likely AI-generated:** our system found strong signs that this text was written with an AI tool (AI-likelihood score: N%). Automated detection can be wrong. If you wrote this yourself, you can appeal this result. |
| 0.35 – 0.65 | **Origin uncertain:** we couldn't tell with confidence whether this text was written by a person or with an AI tool (AI-likelihood score: N%). No action is taken based on this result. |
| < 0.35 | **Likely human-written:** our system found no strong signs of AI generation in this text (AI-likelihood score: N%). |
| too short (score `null`) | **Origin uncertain:** this text is too short to analyze reliably (fewer than 4 sentences), so no AI-likelihood score was given. |

---

## Detection signals
Both signals take the sentence list and return a score on the 0.0–1.0 scale above.

### Signal 1 — `llm_score(sentences)`
- Sends the text to Groq (`qwen/qwen3.8-27b`) with a prompt instructing the model to reply with JSON only, e.g. `{"score": 0.82}`.
- The prompt asks the model to be calibrated: it gives a guide for each score band, keeps scores below 0.15 or above 0.85 for strong evidence only, and asks for 0.35–0.65 when the text is generic or the evidence is mixed. Without this, the model only answered ~0.05–0.15 or ~0.85–1.0, so nothing ever landed in "uncertain" and plain human writing (essays, news-style text) was flagged as AI.
- If the call fails (timeout, rate limit, unparseable reply) → return `None`.
- Weighted **0.5** in the final score.


### Signal 2 — `stylometric_score(sentences)`
Averages several 0.0–1.0 sub-scores:
- **Phrase bank:** count of AI-typical words/phrases ("delve", "tapestry", "it's important to note", "it's not...it's"...), normalized: `min(1.0, hits / len(sentences))`.
- **Sentence length uniformity:** low variation in sentence length → closer to 1.0.
- **Vocabulary variety:** low unique-words ÷ total-words ratio → closer to 1.0.
- *(optional)* **Punctuation patterns:** e.g. heavy em-dash use, no contractions.
- Weighted **0.5** in the final score (or 1.0 if the LLM signal is unavailable).

### Anticipated Edge Cases
- poetry or any piece written by a human in a stylistic way with purposely identical sentence length may be flagged as likely_ai. (Haikus, Shakespeare style iamic pentameter, etc)
- A human written piece whose main topic is about one of the words in the phrase bank may be severely flagged by the second signal. (An expository essay about tapestries for example)

---

## Function summary
| Function | Input | Output |
|---|---|---|
| `text_handler(text)` | `str` | `list[str]` of sentences |
| `length_filter(sentences)` | `list[str]` | `bool` |
| `llm_score(sentences)` | `list[str]` | `float` 0–1, or `None` on failure |
| `stylometric_score(sentences)` | `list[str]` | `float` 0–1 |
| `combine_scores(llm, stylo)` | `float \| None`, `float` | `float` 0–1 |
| `generate_label(score)` | `float \| None` | label `str` |
| `get_response(reason, score=None)` | `"length" \| "score"`, `float` | response `dict` |
| `add_to_log(entry)` | `dict` | — (appends to `log.json`) |
| `find_post(content_id)` | `str` | entry `dict`, or `None` if not found |
| `file_appeal(content_id, reasoning)` | `str`, `str` | `("ok" \| "not_found" \| "already_under_review", entry)` |

---

## AI Tool Plan

### M2
- I gave Claude my brainstorm from M1 and asked it to explain to me the Flask verbaige and error codes that would best work with my ideas. 
- I also gave Claude my handwritten flow chart and asked it to turn it into ASCII art format because I am not the best at it.

### M3 - submission endpoint + first signal
- I will give Claude my Signal section and Diagram and ask it to generate my Flask skeleton and my first Signal.
