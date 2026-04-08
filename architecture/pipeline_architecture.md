# EC FAQ Bot — Full Pipeline Architecture

```mermaid
flowchart TD
    A["User Question (Bengali)"] --> B["Text Cleaning & Unicode Normalization"]
    B --> C{"YES/NO\nShort-circuit?"}
    C -- Yes --> D["Return follow-up answer"]
    C -- No --> E{"Consulate\nAddress Query?"}
    E -- Yes --> F["Return consulate info\n(bypass RAG)"]
    E -- No --> G["E5 Embedding\n(multilingual-e5-large-instruct, 1.1 GB)\n~19 ms/query"]

    G --> H["FAISS Search\n(top-100 candidates from 79K index)"]
    H --> I["Dual-Signal Fusion\n(STS score + Neural Tag Classifier)"]
    I --> J["Ranked Candidates\n(top-3 with scores)"]

    J --> K{"Ambiguity Gate\ntop1 - top2 score gap\n≤ 0.03?"}
    K -- "No (clear winner)" --> L["Use E5/dual-signal result as-is"]
    K -- "Yes (ambiguous)" --> M["SLM Reranker\n(gemma4:e2b via Ollama)\n~1.4 s/query"]

    M --> N["SLM returns JSON:\nbest_index, confidence, abstain"]
    N --> O{"Override Gates\nconfidence ≥ 0.60?\ncandidate score ≥ 0.80?"}
    O -- "Both pass\n(Override)" --> P["Replace E5 answer\nwith SLM pick"]
    O -- "Gate fails\n(Fallback)" --> L
    N -- "abstain=true\nor parse error\n(Abstain)" --> L

    L --> Q["Fraction Resolver\n(context augmentation\nfor ambiguous queries)"]
    P --> Q
    Q --> R["Response Formatter\n(thresholding, follow-ups,\nextensions)"]
    R --> S["Final JSON Response\n(answer, tag, score, telemetry)"]

    style A fill:#e3f2fd,stroke:#1565c0
    style G fill:#e8f4e8,stroke:#2d7d2d
    style H fill:#e8f4e8,stroke:#2d7d2d
    style I fill:#e8f4e8,stroke:#2d7d2d
    style M fill:#fff3cd,stroke:#856404
    style P fill:#d4edda,stroke:#155724
    style L fill:#f8f9fa,stroke:#6c757d
    style S fill:#e3f2fd,stroke:#1565c0
```

## Stage Breakdown

| Stage | Component | File | Latency |
| --- | --- | --- | ---: |
| 1 | Text cleaning & normalization | `e5/text_processing.py` | < 1 ms |
| 2 | YES/NO short-circuit | `e5/handlers/yesno_handler.py` | < 1 ms |
| 3 | Consulate detection | `e5/handlers/consulate_handler.py` | < 1 ms |
| 4 | E5 embedding + FAISS search | `e5/pipeline/rag_system.py` | ~19 ms |
| 5 | Dual-signal fusion | `e5/pipeline/dual_signal_wrapper.py` | < 1 ms |
| 6 | Ambiguity gate | `e5/pipeline/search_stage.py` | < 1 ms |
| 7 | SLM reranker (if triggered) | `e5/pipeline/slm_reranker.py` | ~1,400 ms |
| 8 | Fraction resolver | `e5/pipeline/fraction_resolver.py` | < 1 ms |
| 9 | Response formatter | `e5/handlers/response_formatter.py` | < 1 ms |

**Total latency**: ~19 ms (no reranker) or ~1,420 ms (with reranker). The SLM reranker only fires when the ambiguity gate triggers (top-1 minus top-2 score gap ≤ 0.03).
