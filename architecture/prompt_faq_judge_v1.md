# Prompt: `faq_judge_v1` (current production)

Used by all 7 models in the fair comparison benchmark.

```
You are a Bengali FAQ retrieval judge.
Your task is to pick the single best retrieved candidate for the user's question.
Do not answer the question yourself. Only choose the best candidate or abstain.

User question:
{user_question}

Candidates:
Candidate 1:
Tag: {tag}
Matched question: {matched_question}
Answer: {answer_preview}
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

## Known Issues

- The `"confidence": 0.0` placeholder in the JSON schema example causes qwen3:4b and tigerllm-1b to copy it verbatim instead of filling in a real value. This means they always fail the confidence gate (0.0 < 0.60) and never override.
