# SLM Reranker — Decision Flow & Design

## Reranker Decision Flow

```mermaid
flowchart TD
    A["Top-3 FAISS Candidates\n(with scores)"] --> B{"Ambiguity Gate\ntop1 - top2 gap ≤ 0.03?"}
    B -- "No\n(clear winner)" --> C["Skip reranker\nUse E5 result"]
    B -- "Yes\n(ambiguous)" --> D["Build Judge Prompt\nfaq_judge_v1"]

    D --> E["Progressive Compaction\n(5 levels, fit within 3712 token budget)"]
    E --> F["POST to Ollama\nhttp://127.0.0.1:11434/api/generate\ngemma4:e2b, temp=0.0, num_predict=128"]
    F --> G{"Parse JSON\nResponse?"}

    G -- "Parse failed\nor empty" --> H["ABSTAIN\nreason: invalid_json\nor request_failed"]
    G -- "Valid JSON" --> I{"abstain == true\nor best_index == null?"}

    I -- Yes --> J["ABSTAIN\nreason: model_abstained"]
    I -- No --> K["Extract:\nbest_index, confidence"]

    K --> L{"Confidence Gate\nconfidence ≥ 0.60?"}
    L -- No --> M["FALLBACK\n(SLM not confident enough)"]
    L -- Yes --> N{"Candidate Score Gate\ncandidate E5 score ≥ 0.80?"}

    N -- No --> O["FALLBACK\n(candidate score too low)"]
    N -- Yes --> P["OVERRIDE\nReplace E5 answer\nwith SLM's pick"]

    H --> Q["Keep original\nE5 answer"]
    J --> Q
    M --> Q
    O --> Q

    P --> R["Use SLM pick\nas final answer"]

    Q --> S["Attach telemetry\n(profile, latency, confidence,\nabstain, reason)"]
    R --> S

    style B fill:#fff3cd,stroke:#856404
    style P fill:#d4edda,stroke:#155724
    style Q fill:#f8f9fa,stroke:#6c757d
    style H fill:#fce4ec,stroke:#c62828
    style J fill:#fce4ec,stroke:#c62828
    style M fill:#fff3cd,stroke:#856404
    style O fill:#fff3cd,stroke:#856404
```

## Three Possible Outcomes

| Outcome | When | Final Answer | Telemetry |
| --- | --- | --- | --- |
| **OVERRIDE** | SLM picked a candidate, confidence ≥ 0.60, candidate score ≥ 0.80 | SLM's pick | `should_apply=true` |
| **FALLBACK** | SLM picked but failed a gate (low confidence or low candidate score) | Original E5 | `should_apply=false, abstain=false` |
| **ABSTAIN** | SLM returned `abstain=true`, empty JSON, or request failed | Original E5 | `should_apply=false, abstain=true` |

## Prompt Template (`faq_judge_v1`)

```
You are a Bengali FAQ retrieval judge.
Your task is to pick the single best retrieved candidate for the user's question.
Do not answer the question yourself. Only choose the best candidate or abstain.

[Recent conversation context (if available):]
[- user: {message}]
[- assistant (tag=X): {message}]

User question:
{user_question (max 280 chars, progressively compacted)}

Candidates:
Candidate 1:
Tag: {tag}
Matched question: {matched_question (max 220 chars)}
Answer: {answer_preview (max 180 chars)}
Retrieval score: {score}

Candidate 2:
...

Candidate 3:
...

Return JSON only with this exact schema:
{
  "best_index": 1,
  "confidence": 0.0,
  "abstain": false
}

Rules:
- best_index must be an integer between 1 and the number of candidates.
- confidence must be a number between 0 and 1.
- If no candidate clearly matches, set abstain to true.
- Judge Bengali meaning and user intent, not surface word overlap.
- Prefer the most specific candidate that directly answers the user's question.
- Do not include markdown, explanations, or extra text.
```

## Prompt Token Budgeting

Total budget: `num_ctx (4096) - num_predict (128) - margin (256) = 3712 tokens`

If the prompt exceeds the budget, it is progressively compacted through 5 levels:

```mermaid
flowchart LR
    L0["Level 0\nQ:280 A:180\n4 context turns"] -- "Over budget" --> L1["Level 1\nQ:240 A:140\n3 turns"]
    L1 -- "Over budget" --> L2["Level 2\nQ:220 A:110\n2 turns"]
    L2 -- "Over budget" --> L3["Level 3\nQ:180 A:80\n1 turn"]
    L3 -- "Over budget" --> L4["Level 4\nQ:160 A:60\n0 turns"]

    style L0 fill:#e8f4e8,stroke:#2d7d2d
    style L4 fill:#fce4ec,stroke:#c62828
```

| Level | Question | Candidate Q | Answer | Context Turns | Message |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 280 chars | 220 chars | 180 chars | 4 | 180 chars |
| 1 | 240 chars | 180 chars | 140 chars | 3 | 140 chars |
| 2 | 220 chars | 160 chars | 110 chars | 2 | 110 chars |
| 3 | 180 chars | 140 chars | 80 chars | 1 | 90 chars |
| 4 | 160 chars | 120 chars | 60 chars | 0 | 80 chars |

## Model Behavior Comparison

```mermaid
flowchart TD
    PROMPT["Same faq_judge_v1 prompt\nSame 3 candidates"] --> G4["gemma4:e2b\n7.2 GB (2-bit)"]
    PROMPT --> G4L["gemma4:latest\n9.6 GB (4-bit)"]
    PROMPT --> TIT["titulm-gemma-2b\n1.7 GB"]
    PROMPT --> GM3["gemma3:1b\n815 MB"]
    PROMPT --> QW4["qwen3:4b\n2.5 GB"]
    PROMPT --> TIG["tigerllm-1b\n806 MB"]
    PROMPT --> QW1["qwen3:1.7b\n1.4 GB"]

    G4 --> G4R["best_index=1\nconfidence=1.0\nabstain=false"]
    G4L --> G4LR["best_index=1\nconfidence=0.95\nabstain=false"]
    TIT --> TITR["best_index=1\nconfidence=0.99\nabstain=false"]
    GM3 --> GM3R["best_index=1\nconfidence=0.99\nabstain=false"]
    QW4 --> QW4R["best_index=1\nconfidence=0.0\nabstain=false"]
    TIG --> TIGR["best_index=1\nconfidence=0.0\nabstain=false"]
    QW1 --> QW1R["{ }"]

    G4R --> OVR["OVERRIDE"]
    G4LR --> OVR
    TITR --> OVR
    GM3R --> OVR
    QW4R --> FALL["FALLBACK\n(conf 0.0 < 0.60)"]
    TIGR --> FALL
    QW1R --> ABS["ABSTAIN\n(empty JSON)"]

    style G4 fill:#e2efda,stroke:#155724
    style G4L fill:#e2efda,stroke:#155724
    style TIT fill:#e2efda,stroke:#155724
    style GM3 fill:#e2efda,stroke:#155724
    style QW4 fill:#fff2cc,stroke:#856404
    style TIG fill:#fff2cc,stroke:#856404
    style QW1 fill:#fce4ec,stroke:#c62828
    style OVR fill:#d4edda,stroke:#155724
    style FALL fill:#fff3cd,stroke:#856404
    style ABS fill:#fce4ec,stroke:#c62828
```

## Ollama Configuration

```
Model:        gemma4:e2b (production recommendation)
Backend:      Ollama (http://127.0.0.1:11434/api/generate)
temperature:  0.0 (deterministic)
num_predict:  128 (max output tokens)
num_ctx:      4096 (context window)
timeout:      90 seconds
```

## Production Gates Summary

The reranker has **three layers of safety** preventing bad overrides:

1. **Ambiguity gate** (pre-SLM): Only invoke SLM when top-1 and top-2 scores are within 0.03. Clear winners skip the SLM entirely — no latency added.
2. **Confidence gate** (post-SLM): SLM must report confidence ≥ 0.60. Low-confidence picks are discarded.
3. **Candidate score gate** (post-SLM): The picked candidate's E5 retrieval score must be ≥ 0.80. Ensures the SLM only picks from reasonably good candidates.

If any gate fails, the original E5 answer is preserved unchanged.
