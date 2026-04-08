# EC FAQ Bot — SLM Reranker Benchmark

Benchmark results for 7 small language models evaluated as post-retrieval rerankers for the Bangladesh Election Commission FAQ Bot. All models judged on 1,128 Bengali questions (700 non-problematic + 428 problematic) using the same `faq_judge_v1` prompt.

## Quick Results

| Model | Size | NP 700 | P 428 | Total | Net vs Vanilla | Mean/Query |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| **gemma4:e2b (2-bit)** | 7.2 GB | 658/700 | 151/428 | **809** | **+109** | 1,430 ms |
| gemma4:latest (4-bit) | 9.6 GB | 625/700 | 179/428 | 804 | +104 | 1,555 ms |
| qwen3:4b | 2.5 GB | 700/700 | 27/428 | 727 | +27 | 2,651 ms |
| gemma3:1b | 815 MB | 700/700 | 22/428 | 722 | +22 | 2,093 ms |
| titulm-gemma-2b:q4km | 1.7 GB | 700/700 | 20/428 | 720 | +20 | 5,040 ms |
| tigerllm-1b:q4km | 806 MB | 700/700 | 20/428 | 720 | +20 | 2,470 ms |
| vanilla (E5 only) | 1.1 GB | 700/700 | 0/428 | 700 | 0 | 19 ms |
| qwen3:1.7b | 1.4 GB | 354/700 | 35/428 | 389 | -311 | 2,239 ms |

**Production recommendation**: gemma4:e2b (2-bit) — best net accuracy (+109), fastest SLM (1.4s/query), fits comfortably on T4 with E5+FAISS.

## Repository Structure

```
ec-faq-bot-benchmark/
├── README.md                          ← this file
├── architecture/
│   ├── pipeline_architecture.md       ← full pipeline Mermaid diagram (E5 → FAISS → SLM → response)
│   ├── slm_reranker_design.md         ← SLM decision flow, prompt template, token budgeting, model behaviors
│   └── benchmark_report_1128.md       ← complete benchmark report with all tables and findings
└── xlsx/
    ├── inference_speed_1128.xlsx       ← accuracy + latency for all 7 models × 2 datasets
    ├── model_prompt_capability.xlsx    ← model capability tiers (capable/partial/broken)
    ├── nonproblematic_700_dataset.xlsx ← 700 questions where E5 is correct
    ├── problematic_428_dataset.xlsx    ← 428 questions where E5 fails with high confidence
    ├── full_dataset_question_tag.xlsx  ← complete 79,214 question-tag dataset
    └── next_sprint_tasks.xlsx         ← 34 Jira-ready tasks across 6 epics
```

## Architecture

### Full Pipeline

User question flows through: text normalization → YES/NO handler → consulate detector → **E5 embedding + FAISS search** (top-100, ~19ms) → dual-signal fusion → ambiguity gate → **SLM reranker** (if gap ≤ 0.03, ~1.4s) → response formatter.

See [`architecture/pipeline_architecture.md`](architecture/pipeline_architecture.md) for the full Mermaid diagram.

### SLM Reranker

**What is a judge?** The SLM (Small Language Model) acts as a **judge** — it does not generate answers. It receives the user's Bengali question and 3 candidate answers retrieved by FAISS, reads them, and picks the best one. It outputs structured JSON: `{"best_index": 1, "confidence": 0.95, "abstain": false}`. The judge never writes its own response — the final answer always comes from the pre-existing FAQ dataset. The SLM only decides *which* pre-existing answer to show.

**What is a gate?** A gate is a **threshold check** that must pass before the system acts on the SLM's decision. Gates are safety mechanisms — they prevent the SLM from overriding a correct E5 answer with a bad pick. If any gate fails, the system silently ignores the SLM's pick and keeps the original E5 answer. The user never knows the SLM was consulted.

The reranker has three gates:
1. **Ambiguity gate** (pre-SLM): only invokes the SLM when top-1 and top-2 FAISS scores are within 0.03. If the retrieval result is clear (large score gap), the SLM is never called — no latency added.
2. **Confidence gate** (post-SLM): the SLM must self-report confidence ≥ 0.60 in its JSON output. If it says confidence=0.0 (like tigerllm always does), the pick is discarded.
3. **Candidate score gate** (post-SLM): the picked candidate's original E5 retrieval score must be ≥ 0.80. Prevents the SLM from selecting a low-quality candidate that FAISS ranked poorly.

Three possible outcomes:

| Metric | What it means | When it happens |
| --- | --- | --- |
| **Override Rate** | Fraction of queries where the SLM's pick **replaced** the original E5 answer. | SLM picked a candidate **AND** its self-reported confidence ≥ 0.60 **AND** the candidate's E5 retrieval score ≥ 0.80. All three conditions must pass. |
| **Abstain Rate** | Fraction of queries where the SLM **refused to judge** — returned `"abstain": true`, empty JSON `{}`, or the request failed/timed out. | System falls back to the original E5 answer unchanged. This is a model-level limitation (e.g. qwen3:1.7b returns empty JSON ~45% of the time because it can't process the prompt). |
| **Fallback Rate** | Fraction of queries where the SLM **picked a candidate but failed a gate** — either confidence was too low or candidate score was too low. Implicit: `1 - Override Rate - Abstain Rate`. | System discards the SLM's pick and keeps the original E5 answer. Example: tigerllm-1b always picks Candidate 1 but reports confidence=0.0 (copies the placeholder from the prompt schema), so it has 0% override and 100% fallback. |

In all non-override cases (fallback or abstain), the final answer is the **original E5 retrieval result** — the SLM never makes things worse, it either improves or is ignored.

See [`architecture/slm_reranker_design.md`](architecture/slm_reranker_design.md) for the decision flow diagram, prompt template, and model behavior comparison.

## Model Prompt Capability

| Tier | Models | Behavior |
| --- | --- | --- |
| **Fully capable** | gemma4:e2b, gemma4:latest, titulm-gemma-2b, gemma3:1b | Valid JSON, real confidence values |
| **Partial** | qwen3:4b, tigerllm-1b | Valid JSON but copies placeholder confidence (0.0) |
| **Broken** | qwen3:1.7b | Returns empty JSON ~45% of the time (model limitation, not system issue) |

## Hardware

- **GPU**: Tesla T4, 16 GB VRAM
- **Retrieval**: multilingual-e5-large-instruct (1.1 GB) + FAISS GPU index
- **SLM inference**: Ollama (local, temperature=0.0, deterministic)
- **Dataset**: 79,214 Bengali FAQ questions, 460 tags, 471 answers
- **Domain**: Bangladesh Election Commission (voter registration, NID, postal voting)

## Key Findings

1. **gemma4:e2b wins** — +109 net, fastest, smallest VRAM footprint among capable models
2. **4-bit is not better than 2-bit** — gemma4:latest fixes 28 more hard cases but breaks 33 more easy ones (net +104 vs +109), and is 2.4 GB larger
3. **4 models achieve 100% on non-problematic** but barely fix problematic queries (+20 to +27)
4. **tigerllm-1b is a no-op** — 0% override rate, copies placeholder confidence, never commits
5. **qwen3:1.7b is broken** — can't reliably produce JSON, destroys non-problematic accuracy
6. **E5 retrieval is not the bottleneck** — 19 ms vs 1,400–5,000 ms for SLM
