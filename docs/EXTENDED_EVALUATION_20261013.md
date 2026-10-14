# Extended Evaluation of a Qwen 27B Domain-Adapted Candidate — 2026-10-13

> **Public, de-identified research summary.** This report uses **aggregated measurements** from archived internal tests. It contains no enterprise prompts, SQL, business outputs, training examples, member identifiers, service endpoints, or internal configuration. The original environment and evaluator are not available in this public package, so the figures **cannot be independently reproduced from this repository**. Published with project-owner authorization for the de-identified summary.
>
> **Status:** Engineering case study / candidate for internal trial. **Not** a Production Release; **not** a general-purpose model benchmark.

[View the self-contained dashboard source](public-dashboard/index.html) · [Download the aggregate-only JSON](public-dashboard/aggregate_metrics.json) · [Existing evaluation methodology](EVALUATION_METHOD.md)

## Executive Summary

This follow-up study looked beyond a small fixed-gate evaluation. Its purpose was to determine whether an adapted 27B model on **one NVIDIA DGX Spark / GB10 (128 GB unified memory)** could retain domain-specific constraints while supporting tool-assisted analysis, long inputs, and multiple concurrent generation streams. The candidate used an SGLang-based inference stack with DFlash2 speculative decoding.

The quality suites exercised an **application-level assistant configuration** that included system instructions, retrieved context and, on some cases, data-tool calls. The results therefore **do not isolate QLoRA model quality from RAG, prompting or tools**. The archived 2026-10-13 run recorded **37 PASS / 2 PARTIAL / 0 FAIL** on a 39-case acceptance suite, and **91 PASS / 4 PARTIAL / 5 FAIL** on a 100-case development set. These are **one-reviewer rubric judgments**, not independently audited accuracy rates. In particular, the development set is near-distribution rather than a clean holdout. The 134-case final holdout was **not** scored and has five known verbatim training-question overlaps that must be resolved before use.

For fixed-length, reasoning-OFF generation, recorded **aggregate** throughput reached **110.8 tokens/s at eight streams**, but the 32-stream result of **145.7 tokens/s** came with a worst observed time-to-first-token (TTFT) of **83.32 s**. This is a queueing/latency trade-off, **not a claim that any individual user achieved 110+ tokens/s**.

**繁體中文重點：** 這次補充了較完整的品質分級、工具使用後的答覆風險、長上下文與並發測試。37/39、91/100 是封存單一評分者依準則分類的結果，不等於通用模型準確率。8 路 110.8 tok/s 是**整體吞吐**，不代表單人速度或正式多人使用容量。

## 1. Evaluation Scope and Evidence

| Category | Archived result | What it establishes — and does not establish |
|---|---:|---|
| Fixed knowledge gate | 5/5 | A small fixed-check pass, not general knowledge accuracy |
| Fixed tool workflow gate | 6/6 | Recorded behavior on six task-specific tool checks; not a security or integration certification |
| In-depth analysis gate | 8/8 **completed** | Eight final reports were produced; **not** eight independently correct analyses |
| Acceptance suite (39 cases) | **37 PASS / 2 PARTIAL / 0 FAIL** | One AI-assisted reviewer checked semantic, operational, recovery and governance scenarios against archived expectations |
| Development suite (100 cases) | **91 PASS / 4 PARTIAL / 5 FAIL** | Near-distribution development evaluation with a single reviewer |
| Continuous multi-turn (10 turns) | **8 PASS / 2 PARTIAL** | One short conversation; does not establish long-session stability |
| General tasks without the production system prompt | **20/24** | Limited check for capability drift and response-format problems |

**Grading method and unit of evaluation.** The larger graded suites evaluated the full assistant workflow (system instructions, retrieval and possible data tools), not base-model or adapter-only accuracy. The fixed gates used their original rule-based or exact-output checks. The 39-case and 100-case suites were classified **PASS / PARTIAL / FAIL by a single Claude-assisted reviewer** against expected outcomes, with no independent second reviewer. The rubric and underlying proprietary prompts/answers are not distributed. Counts are therefore reported as *recorded grades*, not as statistically robust model accuracy.

**Data separation.** The development questions were created from related data patterns. An archived similarity analysis found **23/100** development questions with a highest training-question similarity of at least 0.7; this measure does not independently rule out broader conceptual overlap. The final 134-question holdout was not evaluated, and **five questions had verbatim question-text overlap** with training examples. No clean-held-out generalization or superiority claim is supported.

## 2. Tool-Assisted Workflows: the Important Failure Pattern

The 100 development cases split by whether the agent invoked a data tool:

| Development subset | Total | PASS | PARTIAL | FAIL | Non-PASS |
|---|---:|---:|---:|---:|---:|
| No data-tool call | 72 | 70 | 1 | 1 | 2 / 72 |
| With data-tool call | 28 | 21 | 3 | 4 | **7 / 28** |

The key engineering concern was **answer-policy drift after tool output**. In several recorded cases, the agent used newly retrieved data to overrule a canonical domain definition, inferred an answer from an insufficient condition, or returned unmasked identifiers in response to a request for examples. These are generalized failure categories; no real queries, values, identifiers or business definitions are reproduced here.

**Interpretation:** The association is real within this small archived suite, but **it is not a causal or controlled comparison**; cases that call tools can differ in difficulty from cases that do not. A sensible next evaluation would separate tool selection, data interpretation, response policy compliance and identifier redaction into individually scored gates.

## 3. Decode Throughput vs. Concurrent-Service Latency

The fixed speed gate was run twice using the original prompt construction and streaming decode metric, excluding TTFT:

| Run | Recorded decode throughput |
|---|---:|
| Run 1 | **39.93 tok/s** |
| Run 2 | **38.95 tok/s** |

These figures belong to that specific eight-prompt gate and should not be compared directly to the artificial fixed-output concurrency test below. Neither run isolates the contribution of QLoRA, quantization, runtime selection, or speculative decoding.

The concurrency exercise used **one machine**, a fixed **512 output tokens per request**, greedy generation, reasoning disabled, and forced completion to the output limit. Every number in the *Aggregate throughput* column is the **combined output of all streams**.

| Streams | Aggregate throughput | Average per-stream decode | Slowest per-stream decode | Worst TTFT |
|---:|---:|---:|---:|---:|
| 1 | 24.2 tok/s | 24.37 tok/s | 24.37 tok/s | 0.16 s |
| 2 | 46.2 tok/s | 24.29 tok/s | 23.35 tok/s | 0.27 s |
| 4 | 73.8 tok/s | 20.20 tok/s | 18.58 tok/s | 0.26 s |
| **8** | **110.8 tok/s** | **17.31 tok/s** | **13.93 tok/s** | **0.30 s** |
| 16 | 125.2 tok/s | 13.06 tok/s | 9.28 tok/s | **32.93 s** |
| 32 | 145.7 tok/s | 12.57 tok/s | 8.93 tok/s | **83.32 s** |

**Engineering lesson:** Aggregate tokens/s may continue rising while user-perceived responsiveness deteriorates. Above eight streams, this test recorded substantial worst-case first-token delay. It does **not** establish an eight-user production capacity: real enterprise sessions have variable prompt lengths, reasoning tokens, tool calls, RAG retrieval, streaming behavior, and longer interactions.

## 4. Long-Context Retrieval Is Not Long-Context Reasoning

A synthetic single-needle lookup was tested at each input size, with cache-avoidance measures in the archived procedure. The needle was retrieved in **7/7** runs, but TTFT grew sharply:

| Input tokens | TTFT | Post-first-token decode | Needle located |
|---:|---:|---:|---|
| 1,136 | 0.47 s | 15.16 tok/s | Yes |
| 3,904 | 1.57 s | 18.59 tok/s | Yes |
| 16,499 | 7.23 s | 16.25 tok/s | Yes |
| 30,828 | 15.07 s | 15.54 tok/s | Yes |
| 60,848 | 36.94 s | 15.54 tok/s | Yes |
| 114,635 | 93.69 s | 14.41 tok/s | Yes |
| **170,979** | **178.61 s** | **13.77 tok/s** | **Yes** |

This was a **single retrieval probe** at each input size, not proof of reliable reasoning, synthesis, or multi-document attribution over 171k tokens. A successful needle response after nearly three minutes illustrates the difference between a technical context-window capability and a usable product experience.

## 5. Operational and Quality Risks That Remain Open

1. **Tool-output authority:** Canonical constraints must remain authoritative when retrieved records seem to imply a different rule. Add explicit policy-adherence tests after data retrieval.
2. **Identifier disclosure:** At least one development-case response returned non-public identifiers rather than masking them. The values are excluded; add a dedicated redaction gate.
3. **Reasoning-only / missing visible answer:** Some requests without the normal application prompt exhausted an output budget in reasoning or ended without a visible answer. Parser behavior and final-answer completion need explicit acceptance tests.
4. **Response-style regression:** A reasoning-OFF assistant sometimes produced over-short writing responses, even when it handled short factual questions adequately.
5. **Language consistency:** Mixed simplified/traditional characters were observed in some Chinese responses. The issue was workload- and prompt-dependent; this is not a measured general error rate.
6. **Evaluation contamination:** Rebuild or deduplicate the 134-case holdout before using it, then add independent blinded review and reporting of disagreements.
7. **Long-running tasks:** The repeat depth gate produced 8/8 final reports but averaged **406.4 seconds** per report. Final-answer completion, factual correctness, and completion time must be tracked separately.

None of these findings demonstrates a vulnerability in a named organization's deployed systems; they are observations from an **internal, not-production engineering study**.

## 6. Proposed Reproducibility Protocol for Future Work

For others evaluating a comparable on-premises model, a useful **proposed** protocol is to:

1. Publish a synthetic task taxonomy and a versioned benchmark manifest (prompt hashes, system-prompt presence, model/runtime versions, decoding settings, dataset provenance).
2. Keep a held-out evaluation set separate from teacher generation and training, then run exact-text and near-duplicate checks before grading.
3. Evaluate tool-use decisions, answer correctness, sensitive-identifier handling, and policy constraints as separate measurements.
4. Log response phases (reasoning vs. answer), TTFT, output tokens, complete task latency, and speculative decoding acceptance as different metrics.
5. Test one-stream throughput and concurrent total throughput **alongside tail TTFT and slowest-request speed**, using identical request-length controls.
6. Repeat each condition, specify sample count and cache state, and report observed variance instead of ranking models from a single run.
7. Conduct independent human reviews for borderline or high-impact answers, with inter-rater disagreement recorded.

These are *recommendations*, **not** tests already completed by this project. The public JSON contains only manually whitelisted aggregate numbers; the corresponding sanitized HTML dashboard is a new presentation layer, not a reproduction of the private dashboard or its underlying data.

## 7. Scope, Authorship and Publication Boundary

The project owner directed enterprise requirements, experiment priorities, technical decisions, and evaluation acceptance. Claude Code performed most hands-on training, serving setup, tests, and debugging; later documentation and public presentation were developed through AI-assisted workflows. This public report should not be interpreted as the owner having personally authored all experimental code.

Excluded from this public report and dashboard: proprietary question text, prompts, responses, SQL, database and table names, customer or member records, source scripts, raw logs, environment paths, network addresses, API credentials, and original internal dashboards. The aggregated results still originate in a private corporate project: **The project owner authorized publication of this de-identified summary. The archived private source data remain excluded.**