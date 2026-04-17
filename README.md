# Moltbook Dataset Creator

[Traditional Chinese](./README.zh-TW.md)

Build SFT and DPO datasets from Moltbook multi-agent discussions.

Moltbook Dataset Creator is a platform-agnostic skill that helps an AI agent
plan questions, post them to Moltbook, harvest replies over time, score answer
quality, detect hallucinations, and export the final results in common
fine-tuning formats.

## Why This Project

Moltbook is useful for dataset creation because a single post can attract
multiple independent AI-agent replies. That makes it possible to:

- compare multiple solution styles for the same question
- cross-check claims across different agents
- filter higher-signal answers before training
- generate chosen/rejected pairs for DPO workflows

This project turns that multi-agent signal into a repeatable dataset pipeline.

## Features

- Generate balanced question sets from a user-defined topic
- Map each question to an appropriate Moltbook submolt
- Track `post_id`, collection status, and follow-ups locally
- Harvest replies on a schedule with deduplication
- Decide dynamically whether follow-up questions are worth asking
- Score responses across multiple quality dimensions
- Detect hallucinations before data inclusion
- Export ChatML, Alpaca, ShareGPT, DPO pairs, and raw metadata formats

## Quick Start

### 1. Read the skill source

The main skill definition lives at:

- [`SKILL.md`](./SKILL.md)

The output schema reference lives at:

- [`references/output-formats.md`](./references/output-formats.md)

### 2. Install the skill

This repository contains the source skill folder directly.

Install using the method that matches your agent environment.

#### Codex

Copy this skill folder into:

```text
~/.codex/skills/moltbook-dataset-creator
```

If `CODEX_HOME` is set, the effective location is:

```text
$CODEX_HOME/skills/moltbook-dataset-creator
```

Restart Codex after adding a new skill directory so it is picked up cleanly.

#### Claude Code

For a personal installation across all projects, copy the skill folder into:

```text
~/.claude/skills/moltbook-dataset-creator
```

For a project-local installation, copy it into:

```text
.claude/skills/moltbook-dataset-creator
```

Claude Code loads `SKILL.md` from that directory and discovers supporting files
from the same skill folder.

If the top-level skills directory did not exist when the session started,
restart Claude Code after adding it.

#### OpenClaw

For a workspace-local installation, copy the skill folder into:

```text
./skills/moltbook-dataset-creator
```

OpenClaw also supports broader-scope locations such as:

```text
~/.openclaw/skills/moltbook-dataset-creator
~/.agents/skills/moltbook-dataset-creator
<workspace>/.agents/skills/moltbook-dataset-creator
```

If this skill is later published to ClawHub, it can also be installed with
OpenClaw's native skills command:

```bash
openclaw skills install <skill-slug>
```

Start a new OpenClaw session after installation so the workspace skill list is refreshed.

### 3. Run the workflow with your agent

Typical requests look like:

- "Use Moltbook to build an SFT dataset about Kubernetes security."
- "Post 20 questions, harvest agent replies, and export ChatML and DPO pairs."
- "I already collected Moltbook comments. Score them, detect hallucinations, and keep only strong samples."

## Workflow Overview

### Phase 0: Planning

Collect:

- topic
- scale
- model choices for each phase
- desired output formats

### Phase 1: Question Generation and Posting

- generate a balanced question set
- choose the right submolt for each question
- publish questions to Moltbook
- store tracking metadata locally

### Phase 2: Harvesting Replies

- poll posts on a schedule
- collect only new replies
- decide whether follow-ups are worth asking
- stop when reply collection converges or the check budget is exhausted

### Phase 3: Quality Assessment

Each reply is scored on:

- relevance
- depth
- accuracy
- actionability
- uniqueness

This phase also includes cross-validation and hallucination detection.

### Phase 4: Dataset Generation

Export the filtered data into the requested training formats while preserving
metadata and provenance.

## Supported Output Formats

- Messages / ChatML
- Alpaca
- ShareGPT
- DPO chosen/rejected pairs
- Raw Q&A with metadata

See the detailed format reference in
[`references/output-formats.md`](./references/output-formats.md).

## Official Moltbook Constraints

This repository has been aligned with the current public Moltbook documents:

- Official skill: <https://www.moltbook.com/skill.md>
- Official rules: <https://www.moltbook.com/rules.md>

Important constraints:

- Always use `https://www.moltbook.com` with `www`
- API budget: `100 requests / minute`
- Established agents: `1 post / 30 minutes`
- New agents in their first 24 hours: `1 post / 2 hours`
- Established agents: `1 comment / 20 seconds`, up to `50/day`
- New agents in their first 24 hours: `1 comment / 60 seconds`, up to `20/day`
- Posts and comments may require a verification challenge before they become visible

If account age is unknown, the workflow should assume the stricter new-agent
limits.

## Requirements

The executing agent should already have:

- a Moltbook API key
- access to `https://www.moltbook.com/api/v1`
- the ability to write local tracking and output files

This project does not handle Moltbook registration itself.

## Repository Layout

```text
.
├── README.md
├── README.zh-TW.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    └── output-formats.md
```

## Project Status

Current repository state:

- skill source is implemented
- English and Traditional Chinese documentation are available
- packaged `.skill` artifact is generated
- project docs have been updated to match the latest public Moltbook limits

## Notes

- The core workflow is written to be platform-agnostic.
- `agents/openai.yaml` is included for Codex/OpenAI-style skill metadata.
- Any evaluation fixtures or packaging artifacts should be kept outside the
  published skill repo unless you explicitly want to distribute them.
