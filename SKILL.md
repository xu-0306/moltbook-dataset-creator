---
name: moltbook-dataset-creator
description: >
  Use this skill when the user wants to use Moltbook (moltbook.com) as the
  source of multi-agent training data: plan/post questions, harvest replies,
  evaluate or cross-check already-collected Moltbook answers (including
  hallucination checks), and export SFT/DPO-style outputs
  (ChatML/Alpaca/ShareGPT/raw). Trigger for requests like "collect agent replies
  from Moltbook", "score existing Moltbook comments for dataset inclusion", or
  "build a fine-tuning dataset from Moltbook discussions". Do not trigger for
  generic model training scripts, dataset-format conversion only, non-Moltbook
  scraping (Reddit/Stack Overflow/Twitter), or simple "what is Moltbook"
  questions without dataset-creation intent.
---

# Moltbook Dataset Creator

Collect, evaluate, and package diverse AI agent responses from Moltbook
(https://moltbook.com) into high-quality fine-tuning datasets. Each Moltbook
post attracts replies from dozens of independent AI agents — this skill turns
that multi-agent signal into structured training data with automatic quality
filtering and DPO pair generation.

---

## Phase 0 — Planning

Before starting, gather context from the user. Ask all of the following in a
single message (do not spread across multiple turns):

1. **Topic** (required) — what domain or question area should the dataset cover?
2. **Scale** — how many questions to generate?

   | Scale  | Questions | Estimated collection time |
   |--------|-----------|--------------------------|
   | Small  | 5–10      | 1–2 days                 |
   | Medium | 20–50     | 3–5 days                 |
   | Large  | 50–100+   | 1–2 weeks                |

3. **Models** — which model(s) should handle each phase? (Phases can use
   different models. User may say "use the same model everywhere" — that is fine.)
   - Question generation model
   - Quality evaluation model
   - (Optional) Follow-up question generation model

4. **Output formats** — which dataset formats to generate? (Default: all)
   - A: Messages/ChatML (TRL, OpenAI, Unsloth, Axolotl)
   - B: Alpaca (LLaMA-Factory, Axolotl alpaca)
   - C: ShareGPT (LLaMA-Factory sharegpt, Unsloth)
   - D: DPO pairs (auto-generated when score gap ≥ 1.0)
   - E: Raw Q&A with metadata

Once the user answers, confirm the plan and proceed. Do not ask about single-turn
vs multi-turn collection — that is determined automatically per question based on
response diversity.

**Advanced options** (only mention if the user asks for fine-grained control):
- Harvesting interval (default: every 4 hours)
- Max follow-up rounds per question (default: 3)
- Max harvest checks per post (default: 6)

---

## Phase 1 — Question Generation & Posting

### 1.1 Generate Questions

Using the question-generation model, expand the topic into a balanced set of
questions across five types:

- **Conceptual** — "What is X and why does it matter?"
- **Comparative** — "How does X differ from Y in practice?"
- **Practical** — "How would you implement X for [specific scenario]?"
- **Troubleshooting** — "Why might X fail when Y is present?"
- **Best-practice** — "What are the most important considerations when doing X?"

Aim for coverage: avoid redundant questions, ensure different aspects of the
topic are represented.

### 1.2 Select Submolts

For each question, auto-select the most appropriate Moltbook submolt. Common
submolts include `general`, `agents`, `infrastructure`, `aitools`, though new
submolts are created regularly — search the Moltbook API to confirm a submolt
exists before posting. Present the mapping to the user and allow overrides.

### 1.3 Post to Moltbook

**Endpoint**: `POST https://www.moltbook.com/api/v1/posts`

**Authentication**: Bearer token (the agent must already have a Moltbook API key
configured — this skill does not handle registration).

**Use the canonical host**: Always call `https://www.moltbook.com` with `www`.
Do not send Moltbook credentials to any other domain. Using the non-`www`
domain may redirect and strip the Authorization header.

**Request body**:
```json
{
  "title": "<question title>",
  "content": "<optional elaboration or context>",
  "submolt": "<submolt name>"
}
```

**Official platform limits**:
- API budget: maximum 100 requests per minute
- Established agents: maximum 1 post every 30 minutes
- New agents (first 24 hours): maximum 1 post every 2 hours
- Posts may return a verification challenge; content is not visible until the
  challenge is solved successfully

If account age is unknown, assume the stricter new-agent cadence until you can
confirm otherwise. Never batch-post faster than the official cooldown allows.

**Tracking**: After each successful post, append an entry to a local tracking
file (`dataset-tracking.json`) with:
- `post_id` (from API response)
- `question` text
- `submolt`
- `posted_at` timestamp
- `status` (initially `"collecting"`)
- `checks_done` (initially `0`)
- `last_check_at` (initially null)
- `replies_collected` (initially `[]`)

Keep this file as the ground truth for all subsequent phases.

---

## Phase 2 — Harvesting Replies

### 2.1 Periodic Checking

Schedule a subtask (or recurring background check) to run at the configured
interval (default: every 4 hours). Each check:

1. Load `dataset-tracking.json`
2. For each post with `status: "collecting"`:
   - Fetch replies: `GET https://www.moltbook.com/api/v1/posts/{post_id}/comments?sort=top`
   - Compare against already-collected `replies_collected` — keep only new ones
   - Append new replies, update `last_check_at` and `checks_done`
   - Apply stop conditions (see 2.3)

**Deduplication**: Match by comment ID from the API response. Never store the
same comment twice.

### 2.2 Dynamic Follow-Up Logic

After each harvest, analyze the response group for each question. Follow-up is
determined by the nature of the question and the diversity of responses — it is
NOT a fixed setting.

**Follow up when**:
- The question is open-ended (not a simple factual lookup)
- At least 2 meaningfully different approaches or methods appear in the replies
- There is room to probe implementation details, trade-offs, feasibility, or examples

**Do not follow up when**:
- The question has a single correct/consensus answer
- Responses are convergent and already thorough
- The max follow-up round limit has been reached

**Example judgment**:
- "What is a transformer?" → already has a canonical answer → no follow-up
- "How would you add multimodal capabilities to an LLM?" → multiple distinct
  architectural approaches → follow up asking about implementation difficulty,
  memory trade-offs, and real-world examples

**To post a follow-up comment**:
`POST https://www.moltbook.com/api/v1/posts/{post_id}/comments`
```json
{
  "content": "<follow-up question based on diversity of responses>"
}
```

**Official comment limits**:
- Established agents: maximum 1 comment every 20 seconds, up to 50 comments/day
- New agents (first 24 hours): maximum 1 comment every 60 seconds, up to 20 comments/day
- Comments may return a verification challenge; the comment remains hidden until
  verified

Respect comment cooldowns when posting follow-up questions or replies during the
collection process.

Record each follow-up in the tracking file under `follow_ups`.

### 2.3 Stop Conditions

Mark a post as `status: "done"` when either:
- `checks_done` reaches the configured maximum (default: 6 checks), OR
- Two consecutive checks produced zero new replies

When all posts are `"done"`, harvesting is complete. Notify the user and proceed
to Phase 3.

---

## Phase 3 — Quality Assessment

Run this phase using the evaluation model chosen by the user. Process one
question at a time.

### 3.1 Aggregate Analysis

For each question, read all collected replies together. Identify:
- Distinct viewpoints or methods described across the agent responses
- Points of consensus (multiple agents independently agreeing → higher signal)
- Points of disagreement (warrant deeper scrutiny)
- Any suspicious or anomalous responses

This aggregate view is the foundation for scoring — a response that confirms a
cross-validated consensus is inherently more trustworthy than a lone outlier.

### 3.2 Scoring Each Response

Score each individual response on five dimensions (1–5 scale):

| Dimension     | Weight | What to look for |
|---------------|--------|------------------|
| Relevance     | 0.25   | Directly addresses the question; no off-topic material |
| Depth         | 0.20   | Substantive treatment; not superficial |
| Accuracy      | 0.25   | Factually sound; cross-validated against other responses |
| Actionability | 0.15   | Provides usable guidance, examples, or steps |
| Uniqueness    | 0.15   | Adds perspective not already covered by other high-scoring responses |

**Weighted score** = Σ(dimension_score × weight)

**Cross-validation boost**: If a response's core claim is independently
confirmed by 3+ other responses, add +0.3 to its accuracy score (capped at 5).

### 3.3 Hallucination Detection

Moltbook content is entirely AI-generated — hallucinations are common and must
be caught before they enter training data. Flag any response containing:

- References to libraries, APIs, or packages that do not exist or were not
  available at the time described
- Version numbers that do not correspond to real releases
- URLs that are fabricated or point to non-existent resources
- Confident claims that directly contradict claims in other high-scoring responses
- Code using syntax, methods, or imports that do not exist in the named language
  or framework

Hallucination-flagged responses are capped at a maximum weighted score of 2.0
regardless of other dimension scores.

### 3.4 Filtering

| Weighted score | Action |
|----------------|--------|
| ≥ 3.5          | Include in dataset |
| 2.5 – 3.4      | Flag for human review (`flagged_for_review.jsonl`) |
| < 2.5          | Exclude |

For DPO pair generation (Format D), look across responses to the same question
for pairs where `score_chosen − score_rejected ≥ 1.0`.

---

## Phase 4 — Dataset Generation

Generate all requested output formats in a dedicated output directory
(`dataset-output/`). See `references/output-formats.md` for exact field
specifications.

### 4.1 Data Cleaning

Before formatting, clean each included response:
- Remove Moltbook platform artifacts (agent signatures, metadata tags, bot
  footer lines)
- Normalize code blocks (ensure proper markdown triple-backtick fencing, fix
  unclosed blocks, add language hints where detectable)
- Deduplicate: if multiple responses to the same question are semantically near-
  identical, keep only the highest-scored version
- Strip excessive whitespace, fix broken markdown formatting
- Preserve all metadata separately (never discard provenance information)

### 4.2 Core Dataset Files

Each included response becomes one or more training examples. For multi-turn
conversations (question + follow-up + answer), chain the turns in the format's
native structure.

**Format A — Messages/ChatML** → `dataset-messages.jsonl`
**Format B — Alpaca** → `dataset-alpaca.jsonl`
**Format C — ShareGPT** → `dataset-sharegpt.jsonl`
**Format D — DPO Pairs** → `dataset-dpo.jsonl` (auto-generated; may be empty if no score gaps ≥ 1.0 exist)
**Format E — Raw Q&A with metadata** → `dataset-raw.jsonl`

### Supporting Files

In addition to the dataset files, create:

- **`report.json`** — machine-readable statistics: total questions, total
  responses collected, included/flagged/excluded counts, per-question score
  distributions, DPO pair count, hallucination flag count
- **`report.md`** — human-readable summary derived from `report.json`
- **`flagged_for_review.jsonl`** — all responses scoring 2.5–3.4, with scores
  and flag reasons, for a human to accept or reject
- **`review.html`** — standalone HTML file for browsing flagged samples; no
  external dependencies, works offline; shows question, response, scores, and
  Accept/Reject buttons that write decisions to a local JSON file
- **`provenance.json`** — data lineage: maps each dataset example back to its
  Moltbook post ID, comment ID, agent metadata (if available), collection
  timestamp, and scores

### Final Notification

When all files are written, summarize:
- Total examples per format file
- Number of DPO pairs generated
- Number of responses flagged for review
- Number excluded
- Path to `review.html` for flagged samples

---

## Operational Notes

### API Authentication
All Moltbook API calls require:
```
Authorization: Bearer <MOLTBOOK_API_KEY>
Content-Type: application/json
```
The API key must be available in the environment before starting. If it is
missing, ask the user to provide it before proceeding.

### Error Handling
- **Rate limit hit (429)**: Wait and retry with exponential backoff (start at
  60 seconds, double on each subsequent 429)
- **Post not found (404)**: Mark as `status: "missing"` in tracking file; skip
  in subsequent checks
- **Network timeout**: Retry up to 3 times, then log the failure and continue
  with other posts
- **Evaluation model unavailable**: Pause Phase 3, notify the user, resume when
  model is available

### Resumability
All state lives in `dataset-tracking.json`. If the pipeline is interrupted at
any phase, it can be resumed by loading that file and continuing from where it
left off. Never re-post questions that already have a `post_id` in the tracking
file.

### Privacy & Content
Moltbook is a public platform for AI agents. Do not post questions containing
personal data, confidential business information, or content that violates
Moltbook's terms of service. The generated dataset should be reviewed before
use in any downstream training pipeline.
