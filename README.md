# hle-gemini-3-0

Per-question results for **DeepWriter (Abraxas 1.5 Engine) using Google Gemini 3.0-Pro** on a text-only slice of the HLE benchmark.

For related article, see: https://deepwriter.com/blog/small-team-beats-worlds-top-ai-labs-at-hle/

Official Site: deepwriter.com

Documentation: docs.deepwriter.com

---

## 1. Purpose

- Provide the raw, per-question outcomes for one HLE run with Gemini 3.0-Pro through DeepWriter.
- Expose the grading logic we used (including how failures were treated).
- Make it easy for others to inspect, audit, or re-analyze the results.

---

## 2. Files in this repository

**Data**

- `questions_and_answer_hle_gem3pro.csv`
- `questions_and_answer_hle_gem3pro.xlsx`  
  (Same content as the CSV, just as an Excel export.)

Each row corresponds to a single HLE question. The final row is a `Totals:` summary row.

Columns:

- `id` – HLE question identifier.
- `question` – full question text.
- `answer` – official / canonical HLE answer.
- `s3_id` – internal storage key used in our pipeline.
- `answer_url` – pointer back to the original item bundle.
- `pdf_timestamp` – time when the question PDF for this run was created.
- `extracted_answer` – answer produced by Gemini 3.0-Pro via DeepWriter.
- `confidence` – model-reported confidence string (e.g. `"95%"`).
- `result` – string label from the auto-grader (`"pass"` / `"fail"`).
- `score` – numeric value actually used in the aggregate HLE score (0 or 1 in this run).
- `tex_timestamp` – time when the TeX/PDF pipeline processed this question.

**Code**

- `benchmark_runner.py` – the internal DeepWriter batch runner & auto-grader used to generate and grade this run.

---

## 3. Dataset and filtering

Starting point:

- 1,000 publicly available HLE questions.

Filtering:

- All **image-based questions were removed**.
- This left **878 text-only questions** in the evaluation set.
- Sorted Alphanumerically by HLE Question ID and run in order.

Every one of these 878 questions appears in the CSV/XLSX files in this repo.

---

## 4. Scoring protocol

High-level flow for each question:

1. DeepWriter submits the question to the Gemini 3.0-Pro model and records the model’s answer as `extracted_answer`.
2. The **official HLE answer** is taken from the public benchmark and stored in `answer`.
3. An **equivalence judge** decides whether the model answer counts as correct:
   - The judge is instructed to check strict mathematical / logical equivalence between `extracted_answer` and `answer`.
   - It ignores superficial formatting / LaTeX differences if the content is the same.
   - There is **no partial credit**: the decision is all-or-nothing.
   - If the judge is uncertain, ambiguous, or cannot parse the result cleanly, the item is treated as **incorrect**.
4. The auto-grader writes:
   - `result` = `"pass"` or `"fail"` (string label),
   - `score` = `1` for correct, `0` for incorrect.

**Failed jobs and infra issues**

- Some jobs never produced a usable answer due to infrastructure or pipeline failures (empty `extracted_answer`).
- **Rule:** every such job is treated as **wrong**.  
  Failed, timed-out, missing, or otherwise broken jobs **receive no credit** and are left in the dataset with `score = 0`.
- In this run there are **4** such cases.

**Human review**

- After the automated run, a human reviewer performed a pass over the aggregate results.

---

## 5. Headline results

On the 878-question text-only subset:

- **Total questions:** 878  
- **Pipeline failures (no usable model answer):** 4  
  - All 4 are counted as incorrect with `score = 0`; no credit given.
- **Questions with non-empty model answers:** 874  
- **Final “correct” count (sum of `score` over all 878 rows):** 447

HLE score for this run:

> **50.911** (447 / 878 × 100)

A human reviewed the final scores and the aggregate numbers before publishing this repository.

---

## 6. Auto-grader details (`benchmark_runner.py`)

For readers who want to inspect the grading logic more closely:

- The batch runner:
  - Talks to the DeepWriter API, submits questions in batches, and waits for completion.
  - Enforces **strict batch gating**: the next batch does not start until the current one has finished, including grading.
  - Uses a resumable CSV update pattern: rows are updated atomically with retries.
- Grading:
  - Full TeX for each question/answer pair is passed to an LLM-based equivalence judge.
  - The prompt instructs the judge to:
    - Focus solely on mathematical / logical equivalence,
    - Ignore notation differences,
    - Provide a single JSON field `"result"` with `"pass"` or `"fail"`,
    - Default to **`fail`** in ambiguous or error cases.
  - If the judge itself fails repeatedly or returns malformed output, the question is marked as **failed** and receives `score = 0`.

Again, this script is here purely so others can read the exact rules we used; it is not designed as a general-purpose tool.

Logically equivalent: 1a + 2b == 2b + 1a, so long as the order was not part of the challenge.

Not logically equivalent: 1.000000000000000000000000000000000000000000000000001 and 1.

---

## HLE score visualizations

(hle_scores_agentic_comparison.png)
