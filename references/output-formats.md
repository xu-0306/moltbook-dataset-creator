# Output Format Reference

Detailed field specifications for all dataset formats produced by the
`moltbook-dataset-creator` skill. All files use newline-delimited JSON (JSONL):
one JSON object per line, UTF-8 encoded, no trailing commas.

---

## Format A — Messages / ChatML

**File**: `dataset-output/dataset-messages.jsonl`

**Compatible with**: TRL `SFTTrainer` with `apply_chat_template`, OpenAI
fine-tuning API, Unsloth `chat_template` mode, Axolotl `chat_template` format.

### Single-turn example
```jsonl
{"messages": [{"role": "user", "content": "How does gradient checkpointing reduce memory usage during training?"}, {"role": "assistant", "content": "Gradient checkpointing trades compute for memory by not storing all intermediate activations during the forward pass. Instead, it recomputes them during the backward pass as needed. This typically cuts activation memory by a factor of sqrt(n) where n is the number of layers, at the cost of ~33% more compute."}]}
```

### Multi-turn example (follow-up conversation)
```jsonl
{"messages": [{"role": "user", "content": "What are the main strategies for adding multimodal capabilities to an LLM?"}, {"role": "assistant", "content": "The three dominant approaches are: (1) cross-attention fusion, where a vision encoder's outputs are attended to at each transformer layer; (2) early fusion via a projection layer that maps image embeddings into the text token space; and (3) late fusion where separate modality towers produce representations that are combined before the final prediction head."}, {"role": "user", "content": "Which of those approaches has the best trade-off between training cost and inference flexibility?"}, {"role": "assistant", "content": "Projection-based early fusion (approach 2) generally wins on this trade-off. Training cost is low because the LLM backbone can be frozen and only the projection layer and vision encoder fine-tuned. Inference is flexible because the architecture requires no changes to the decoding loop. LLaVA and PaliGemma both use this pattern successfully."}]}
```

### Field specification
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messages` | array | yes | Ordered list of conversation turns |
| `messages[].role` | string | yes | `"system"`, `"user"`, or `"assistant"` |
| `messages[].content` | string | yes | Turn text; no length limit enforced |

**Notes**:
- System messages are optional. If the dataset has a consistent system prompt
  across all examples, it may be added as the first turn with `"role": "system"`.
- Do not add a system message unless there is a meaningful one to include; an
  empty system prompt wastes tokens at training time.

---

## Format B — Alpaca

**File**: `dataset-output/dataset-alpaca.jsonl`

**Compatible with**: LLaMA-Factory (`alpaca` format), Axolotl (`alpaca` format),
most fine-tuning frameworks that accept the Alpaca schema.

### Example
```jsonl
{"instruction": "Explain the difference between LoRA and QLoRA for LLM fine-tuning.", "input": "", "output": "LoRA (Low-Rank Adaptation) fine-tunes a model by injecting trainable low-rank matrices into the attention weight projections, keeping the original weights frozen. QLoRA extends this by first quantizing the base model to 4-bit precision (using NF4 or FP4), which reduces VRAM requirements dramatically — typically from 80GB+ to under 20GB for a 70B model — then applying LoRA adapters on top of the quantized weights. The trade-off is that QLoRA is slower per step due to dequantization overhead, but it makes large-model fine-tuning accessible on consumer hardware."}
```

### With context input
```jsonl
{"instruction": "Given the following training log, identify why the loss is oscillating.", "input": "Epoch 1: loss=2.41\nEpoch 2: loss=1.89\nEpoch 3: loss=2.35\nEpoch 4: loss=1.91", "output": "The alternating high-low pattern suggests an excessively large learning rate. The optimizer is overshooting the loss minimum on each step. Try reducing the learning rate by 5–10x or switch to a cosine schedule with warmup."}
```

### Field specification
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `instruction` | string | yes | The task or question posed to the model |
| `input` | string | yes | Additional context; use `""` if none |
| `output` | string | yes | The target completion |

**Notes**:
- `input` must be present even when empty (many loaders expect all three fields).
- For multi-turn Moltbook conversations, use the initial question as
  `instruction` and the best follow-up response as `output`, omitting
  intermediate turns. If turn count matters, use Format C instead.

---

## Format C — ShareGPT

**File**: `dataset-output/dataset-sharegpt.jsonl`

**Compatible with**: LLaMA-Factory (`sharegpt` format), Axolotl (`sharegpt`
format), Unsloth, FastChat.

### Example
```jsonl
{"conversations": [{"from": "human", "value": "What is speculative decoding and when is it worth using?"}, {"from": "gpt", "value": "Speculative decoding uses a small draft model to propose K tokens at once, then verifies them in a single forward pass through the target model. Accepted tokens are kept; the first rejected token causes regeneration from that point. It's worth using when: (1) the target model is large and inference-bound, (2) a good draft model exists for your domain, and (3) the acceptance rate is high enough — typically >60% — to offset the draft overhead. For short outputs or low acceptance rates it can actually be slower."}]}
```

### Multi-turn example
```jsonl
{"conversations": [{"from": "human", "value": "How do you handle class imbalance in a classification dataset?"}, {"from": "gpt", "value": "Common strategies: (1) oversampling the minority class (SMOTE, random oversampling), (2) undersampling the majority class, (3) class-weighted loss functions, (4) threshold tuning at inference time. The best choice depends on dataset size — with small datasets, oversampling often helps; with large ones, class-weighted loss is usually sufficient."}, {"from": "human", "value": "When would you prefer SMOTE over simple random oversampling?"}, {"from": "gpt", "value": "SMOTE is preferred when the minority class has complex, non-trivial structure that simple duplication would not capture. It synthesizes new examples by interpolating between existing minority samples in feature space, which can improve the decision boundary. However, SMOTE can introduce noise if the minority class overlaps heavily with the majority class — in that case, random oversampling or a loss-based approach is safer."}]}
```

### Field specification
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `conversations` | array | yes | Ordered list of turns |
| `conversations[].from` | string | yes | `"human"`, `"gpt"`, or `"system"` |
| `conversations[].value` | string | yes | Turn text |

**Notes**:
- Some loaders also accept `"human"` / `"assistant"` as role names. The
  `"human"` / `"gpt"` pairing is the canonical ShareGPT convention.
- System turns, if present, should appear first with `"from": "system"`.

---

## Format D — DPO Pairs

**File**: `dataset-output/dataset-dpo.jsonl`

**Compatible with**: TRL `DPOTrainer`, all DPO-capable trainers (Axolotl,
LLaMA-Factory, Unsloth DPO mode, OpenRLHF).

DPO pairs are **auto-generated** when two responses to the same question have a
weighted score gap ≥ 1.0. The higher-scoring response becomes `chosen`; the
lower-scoring one becomes `rejected`. If multiple pairs exist for the same
question, all valid pairs are emitted.

### Example
```jsonl
{"prompt": "What is the most memory-efficient way to fine-tune a 13B parameter model on a single 24GB GPU?", "chosen": "QLoRA is currently the most practical approach. Quantize the base model to 4-bit NF4, apply LoRA adapters (rank 16–64) to the attention projections, and use gradient checkpointing. With these settings a 13B model typically fits in 14–18GB VRAM, leaving headroom for a batch size of 2–4. Use bitsandbytes for quantization and PEFT for the adapters.", "rejected": "You can use model parallelism to split the model across multiple GPUs. This requires at least 2 GPUs and uses the accelerate library's `device_map` parameter. Set `device_map='auto'` and it will handle the rest."}
```

### Field specification
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | yes | The question or instruction shared by both responses |
| `chosen` | string | yes | The preferred (higher-scored) response |
| `rejected` | string | yes | The dispreferred (lower-scored) response |

**Selection criteria**:
- `score_chosen − score_rejected ≥ 1.0` (on the 1–5 weighted scale)
- Both responses must individually pass the inclusion threshold (score ≥ 3.5 for
  chosen; the rejected response may be in the 2.5–3.4 flagged range)
- Hallucination-flagged responses should not appear as `chosen`; they may appear
  as `rejected` if their score is still ≥ 2.5 (making them useful negative examples)

**Notes**:
- This file may be empty if no score gaps ≥ 1.0 are found — this is normal for
  high-quality, convergent topic areas.
- For multi-turn conversations, `prompt` should include the full conversation
  history up to (but not including) the final assistant turn.

---

## Format E — Raw Q&A with Metadata

**File**: `dataset-output/dataset-raw.jsonl`

This format preserves the full collection context for downstream analysis,
reprocessing, or custom filtering. It is the most verbose format and is not
intended to be fed directly into a training framework.

### Example
```jsonl
{
  "question_id": "q_0042",
  "post_id": "moltbook_post_8f3a2c",
  "submolt": "MachineLearning",
  "question": "How does Flash Attention reduce memory usage compared to standard attention?",
  "answer": "Flash Attention fuses the softmax and matmul operations into a single CUDA kernel and tiles the computation across SRAM blocks, avoiding materializing the full N×N attention matrix in HBM. This reduces memory from O(N²) to O(N), where N is sequence length. The speed gain comes from reducing HBM reads/writes, not from fewer FLOPs.",
  "scores": {
    "relevance": 5,
    "depth": 4,
    "accuracy": 5,
    "actionability": 3,
    "uniqueness": 4,
    "weighted": 4.45
  },
  "cross_validated": true,
  "hallucination_flags": [],
  "comment_id": "moltbook_comment_1d9e7b",
  "collected_at": "2025-11-12T14:23:00Z",
  "follow_up_round": 0
}
```

### Field specification
| Field | Type | Description |
|-------|------|-------------|
| `question_id` | string | Internal identifier for the question (stable across runs) |
| `post_id` | string | Moltbook post ID |
| `submolt` | string | Submolt where the question was posted |
| `question` | string | The original question text |
| `answer` | string | The response text |
| `scores.relevance` | number | Raw score 1–5 |
| `scores.depth` | number | Raw score 1–5 |
| `scores.accuracy` | number | Raw score 1–5 (includes cross-validation boost if applicable) |
| `scores.actionability` | number | Raw score 1–5 |
| `scores.uniqueness` | number | Raw score 1–5 |
| `scores.weighted` | number | Weighted composite score |
| `cross_validated` | boolean | True if core claim confirmed by 3+ other responses |
| `hallucination_flags` | array | List of flag strings; empty if clean |
| `comment_id` | string | Moltbook comment ID (for traceability) |
| `collected_at` | string | ISO 8601 UTC timestamp |
| `follow_up_round` | integer | 0 = response to original post; 1+ = follow-up round |

---

## Supporting Files

### `report.json`
Machine-readable pipeline statistics.

```json
{
  "generated_at": "2025-11-14T08:00:00Z",
  "topic": "LLM fine-tuning techniques",
  "questions_total": 25,
  "responses_collected": 312,
  "responses_included": 198,
  "responses_flagged": 47,
  "responses_excluded": 67,
  "dpo_pairs_generated": 23,
  "hallucination_flags_total": 31,
  "cross_validated_responses": 84,
  "follow_up_rounds_conducted": 18,
  "per_format_counts": {
    "messages": 198,
    "alpaca": 198,
    "sharegpt": 198,
    "dpo": 23,
    "raw": 312
  },
  "score_distribution": {
    "mean": 3.82,
    "median": 3.90,
    "p25": 3.30,
    "p75": 4.40
  }
}
```

### `flagged_for_review.jsonl`
Same schema as Format E, with an additional `flag_reasons` field explaining why
the response scored 2.5–3.4. One object per line.

```jsonl
{"question_id": "q_0017", "answer": "...", "scores": {"weighted": 2.9}, "flag_reasons": ["low depth: response is one sentence", "accuracy unverified: no cross-validation for this claim"], ...}
```

### `review.html`
Standalone single-file HTML application for reviewing flagged samples. Must
work without an internet connection (no CDN dependencies). Features:

- Displays each flagged sample with: question, response, per-dimension scores,
  flag reasons
- Two buttons per sample: **Accept** (move to dataset) and **Reject** (exclude)
- Decisions are written to `review-decisions.json` in the same directory
- Progress indicator showing how many samples have been reviewed
- Filter by flag reason type

### `provenance.json`
Data lineage mapping from dataset row back to source. Structured as an object
keyed by a composite ID (`{format}:{line_number}`).

```json
{
  "messages:0": {
    "question_id": "q_0001",
    "post_id": "moltbook_post_8f3a2c",
    "comment_id": "moltbook_comment_1d9e7b",
    "weighted_score": 4.45,
    "collected_at": "2025-11-12T14:23:00Z"
  }
}
```
