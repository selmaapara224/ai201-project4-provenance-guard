# ai201-project4-provenance-guard
curl http://localhost:5001/log

remember to talk about fewer "likely-ai" answers 

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


## Detection Signals

Both signals take the sentence list and return a score on the 0.0–1.0 scale above.

### Signal 1 — `llm_score(sentences)`
- Sends the text to Groq (`qwen/qwen3.8-27b`) with a prompt instructing the model to reply with JSON only, e.g. `{"score": 0.82}`.
- The prompt asks the model to be calibrated: it gives a guide for each score band, keeps scores below 0.15 or above 0.85 for strong evidence only, and asks for 0.35–0.65 when the text is generic or the evidence is mixed. Without this, the model only answered ~0.05–0.15 or ~0.85–1.0, so nothing ever landed in "uncertain" and plain human writing (essays, news-style text) was flagged as AI.
- If the call fails (timeout, rate limit, unparseable reply) → return `None`.
- Weighted **0.7** in the final score.


### Signal 2 — `stylometric_score(sentences)`
Averages several 0.0–1.0 sub-scores:
- **Phrase bank:** count of AI-typical words/phrases ("delve", "tapestry", "it's important to note", "it's not...it's"...), normalized: `min(1.0, hits / len(sentences))`.
- **Sentence length uniformity:** low variation in sentence length → closer to 1.0.
- **Vocabulary variety:** low unique-words ÷ total-words ratio → closer to 1.0.
- *(optional)* **Punctuation patterns:** e.g. heavy em-dash use, no contractions.
- Weighted **0.3** in the final score (or 1.0 if the LLM signal is unavailable).

## Confidence Scoring

The final confidence score is calculated by multipling the LLM and Stylometrics scores to their weights (70% and 30% respectively), then adding the results.

  **final = 0.7*llm + 0.3*stylo; if llm is None, final = stylo.**

### Submission Example 1:

  **Text:** "In today's fast-paced world, it is important to note that technology plays a pivotal role in shaping our lives. Furthermore, it fosters innovation across a rich tapestry of industries. Moreover, organizations must leverage robust solutions to navigate this ever-evolving landscape. Ultimately, embracing these changes is crucial for long-term success. In conclusion, the future is bright for those who harness its potential."

  **Attribution:** "likely_ai"

  **Confidence:** 0.859

  **Label:** "Likely AI-generated"

  **llm_score:** 1.0
  **stylo_score:** 0.53
  **combined_score:** 0.859

### Submission Example 2:

  **Text:** "ok so the bus was late AGAIN. stood there like 40 min lol. my sister said just bike but its raining?? whatever, got coffee at least. the barista spelled my name wrong for the third time this week, honestly kinda impressed."

  **Attribution:** "likely_human"

  **Confidence:** 0.108

  **Label:** "Likely human-written"

  **llm_score:** 0.05
  **stylo_score:** 0.244
  **combined_score:** 0.108


## Transparency Label

`generate_label(score)` returns the `label` text shown with every result. The score is shown as a percentage "AI-likelihood score".

| Score | Label |
|---|---|
| > 0.65 | **Likely AI-generated:** our system found strong signs that this text was written with an AI tool (AI-likelihood score: N%). Automated detection can be wrong. If you wrote this yourself, you can appeal this result. |
| 0.35 – 0.65 | **Origin uncertain:** we couldn't tell with confidence whether this text was written by a person or with an AI tool (AI-likelihood score: N%). No action is taken based on this result. |
| < 0.35 | **Likely human-written:** our system found no strong signs of AI generation in this text (AI-likelihood score: N%). |
| too short (score `null`) | **Origin uncertain:** this text is too short to analyze reliably (fewer than 4 sentences), so no AI-likelihood score was given. |

## Rate limiting

Provenance Guard uses [Flask-Limiter](https://flask-limiter.readthedocs.io/) with in-memory storage (`storage_uri="memory://"`). Limits are counted **per client IP** (`get_remote_address`). Only the routes listed below are limited (`default_limits=[]`). When a client goes over a limit, it gets **HTTP 429** with a JSON body that says which limit it hit:

```json
{"error": "Too many requests. Please slow down.", "limit": "10 per 1 minute"}
```

| Endpoint | Limit | Why |
|---|---|---|
| `POST /submit` | **10 per minute; 100 per day** | See below. |
| `POST /appeal` | **5 per minute; 20 per day** | A creator only appeals posts they've already submitted, and each post can only be appealed once (a second appeal returns 409). 20 a day covers a creator disputing every flagged post from a busy day. |
| `GET /log` | none | Read-only, never calls the LLM, and only returns up to 100 entries. |

### 10 per minute / 100 per day for `/submit`

**Realistic use.** A writer submits their own work, reads the label, maybe edits, and sends it again. Each round takes at least a few seconds to read and edit. Even a fast "tweak and resubmit" loop rarely gets past 3–5 submissions a minute. 10 per minute gives about twice that, so an honest user never sees a 429. 100 per day is far more than one person writes and checks in a day. That covers a heavy day: a dozen pieces with several revisions each.

**Stopping abuse.** A script can send hundreds of requests a second. The per-minute cap holds a single IP to one request every 6 seconds on average. The daily cap stops a slow script that stays under the per-minute limit all day. Without it, that script could make 10 × 60 × 24 = 14,400 requests a day. With it, the most is 100.

**Protecting the shared API key.** Each scored submission makes one call to the Groq API using the project's API key. Groq rate-limits that key per minute and per day. Without our own limits, one client could use up the whole quota, and `llm_score` would start failing for everyone. Scoring would then quietly fall back to stylometry only (`llm_unavailable: true`), which is less accurate. Our per-IP limits make sure no single client can use up the shared quota.



### Rate Limit Test Results:

**Input:**

for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5001/submit \
    -H "Content-Type: application/json" \
    -d '{"text": "This is a test submission for rate limit testing purposes.", "creator_id": "test-user"}'
done


**Output:**

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

### Rate Limit limitations
- `memory://` storage resets when the server restarts. It also isn't shared between worker processes, so with N workers the real limit is N times higher. For production, point `storage_uri` at Redis (`redis://...`).
- Limits are per IP. Users behind a shared NAT (a school or office network) share one limit. Someone who controls many IPs can get around it. Keying the limit on an authenticated user ID would fix both problems, but that is beyond the scope of this project.


## Audit Log

### Example Log Entries Generated from Tests
[
  {
    "content_id": "4ffb77d5-f29e-46e0-87e0-785266bd1174",
    "creator_id": "test-user",
    "timestamp": "2026-11-17T23:10:33.637Z",
    "text": "This is a test submission for rate limit testing purposes.",
    "attribution": "uncertain",
    "confidence": null,
    "label": "Origin uncertain: this text is too short to analyze reliably (fewer than 4 sentences), so no AI-likelihood score was given.",
    "llm_score": null,
    "stylo_score": null,
    "combined_score": null,
    "llm_unavailable": false,
    "status": "classified"
  },
  {
    "content_id": "3d1bd5a0-26a2-4b0b-aa7f-b2a8d35aef45",
    "creator_id": "me",
    "timestamp": "2026-11-16T19:14:13.322Z",
    "text": "In today's fast-paced world, it is important to note that technology plays a pivotal role in shaping our lives. Furthermore, it fosters innovation across a rich tapestry of industries. Moreover, organizations must leverage robust solutions to navigate this ever-evolving landscape. In conclusion, the future is bright for those who harness its potential.",
    "attribution": "likely_ai",
    "confidence": 0.856,
    "label": "Likely AI-generated: our system found strong signs that this text was written with an AI tool (AI-likelihood score: 86%). Automated detection can be wrong. If you wrote this yourself, you can appeal this result.",
    "llm_score": 1.0,
    "stylo_score": 0.52,
    "combined_score": 0.856,
    "llm_unavailable": false,
    "status": "under_review",
    "appeal_reasoning": "I wrote this myself from personal experience.",
    "appeal_filed_at": "2026-11-16T19:25:23.247Z"
  },
  {
    "content_id": "cc97845f-6519-41a3-af23-fe91aec625f7",
    "creator_id": "meagain",
    "timestamp": "2026-11-16T19:22:55.965Z",
    "text": "Hi My name is Pinky and I am seven years old. My favorite color is pink which is why people call me Pinky! I love to watch movies with my mommy and ride bikes with my dad on weekends and sometimes after school too. I love learning multiplication and stuff.",
    "attribution": "likely_human",
    "confidence": 0.154,
    "label": "Likely human-written: our system found no strong signs of AI generation in this text (AI-likelihood score: 15%).",
    "llm_score": 0.05,
    "stylo_score": 0.396,
    "combined_score": 0.154,
    "llm_unavailable": false,
  } 
]

  ## Known Limitations

- poetry or any piece written by a human in a stylistic way with purposely identical sentence length may be flagged as likely_ai. (Haikus, Shakespeare style iamic pentameter, etc)
- A human written piece whose main topic is about one of the words in the phrase bank may be severely flagged by the second signal. (An expository essay about tapestries for example)

  ## Spec Reflection

  The spec was extrmely useful when it came to keeping the output consistent when it comes to what the user sees after submissions and appeals. I wanted specific verbaige and it helped plan that in advance. 

  **Divergence from Spec**: The detection signals were
originally weighted 50/50, but that led to too many posts being rated as likely-ai, even when they were written by myself. The weights were changed to be 70 percent LLM score and 30 percent stylometric. This results in less likely-ai scoring and more uncertain, but it is better than a false confident scoring of likely-ai.



  ## AI Usage

  - I gave Claude my hand drawn architecture diagram and asked it to generate an ASCII art version for planning.md, because I have trouble creating ASCII art. I revised some of the spelling.

  - I gave Claude my planning.md and had it generate the implementation of my signals and combining their scores. I changed the weights of the signals after the results were skewed to likely-ai.

  ## Demo Video

  <div style="position: relative; padding-bottom: 62.5%; height: 0;"><iframe src="https://www.loom.com/embed/5fcf086a0a674f1f9ba88a6be25bea3f" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"></iframe></div>