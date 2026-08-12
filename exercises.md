# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Low-stakes questions, where user can verify independently; simple factual answers with obvious grounding. | Any high-stakes domain (legal, medical, financial); when hallucinations could cause harm or legal liability. | Investigate context quality, add grounding guardrails, implement citation verification. |
| Answer Relevance | Conversational follow-ups where partial relevance is acceptable; user intent is clear from context. | Direct factual questions where relevance <0.6 indicates wrong topic coverage; user explicitly asking for X but getting Y. | Review routing logic, improve prompt alignment with user intent, analyze failure patterns. |
| Context Recall | Supplementary questions where partial evidence is acceptable; when gold context is overly broad. | When user asks for complete information (e.g., "list all X") and retrieval misses >40%; core factual gaps. | Improve retriever's recall, expand index, adjust chunking strategy, add query expansion. |
| Context Precision | Large context windows where top-K retrieval is not critical; exploratory queries. | When most relevant info is buried in chunk #10+; user needs immediate answer and irrelevant chunks add noise. | Implement reranking, improve BM25/semantic similarity, prune irrelevant chunks early. |
| Completeness | Open-ended discussions where partial coverage is acceptable; user explicitly asks for "brief overview." | Step-by-step instructions (e.g., how to register), multi-part questions where missing one part = failure. | Expand retrieval coverage, improve generation's completeness checking, add explicit completeness prompt. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
>
> **Experiment Design: Paired Comparison Test**
>
> **Condition A (Baseline):** Cặp answers A-B được trình bày theo thứ tự A trước, B sau.
> **Condition B (Reversed):** Cùng cặp answers, nhưng đổi ngược: B trước, A sau.
>
> **Setup:**
> - Chuẩn bị 50 pairs of answers (cùng question, hai answers khác chất lượng)
> - Mỗi pair xuất hiện hai lần: một lần A-left/B-right, một lần B-left/A-right
> - 4 versions × 50 pairs = 200 total judgments
> - Randomize order của các pairs cho judge
>
> **Measurement:**
> - Tính tỷ lệ answer được chọn là "better" khi nó ở position 1 vs position 2
> - Nếu answer ở position 1 được chọn >55% (thay vì 50%), có position bias
> - Statistical significance test (chi-squared) để xác nhận bias không phải random noise
>
> **Additional Condition (Bonus):** Neutral baseline — cả hai answers giống hệt nhau. Nếu judge vẫn chọn position 1 >50%, đó là pure position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
>
> **1. Normalize by length in rubric:**
> - Explicitly penalize unnecessary verbosity: "Deduct 1 point if answer is significantly longer than needed to answer the question."
> - Add length comparison in evaluation criteria: "Score based on information density, not word count."
>
> **2. Require conciseness in scoring dimensions:**
> - Add explicit "Conciseness" dimension: 5 = "Direct, complete in minimum words" → 1 = "Verbose with redundant information."
> - Make it orthogonal to quality: an answer can be excellent AND concise.
>
> **3. Instruction-based debiasing in prompt:**
> - "IMPORTANT: Longer answers are NOT inherently better. A complete answer in 3 sentences scores higher than a verbose answer in 10 sentences with the same information."
> - "Ignore answer length when scoring correctness and completeness."
>
> **4. Use token-limited comparison:**
> - Pass answers truncated to same length or normalized format
> - Judge scores based only on content within token budget

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
>
> **Vấn đề cốt lõi:** LLM judge không phải ground truth — nó là một model có thể systematic errors mà ta không biết nếu không so với human judgment.
>
> **Lý do cụ thể:**
>
> 1. **Establish reliability:** Calibration xác nhận judge aligns với human interpretation của rubric. Không có calibration, ta không biết "4/5" của judge có nghĩa gì thực sự.
>
> 2. **Detect systematic bias:** Human labels reveal whether judge consistently rates certain categories too high/low (e.g., always overestimates completeness).
>
> 3. **Set meaningful thresholds:** Calibration data cho phép map raw LLM scores → human-interpreted quality levels. Không có calibration, threshold 0.8 có thể quá strict hoặc quá lenient.
>
> 4. **Validate for domain:** LLM có training biases. Human calibration确保 judge hiểu domain-specific standards (e.g., legal precision vs. casual tone).
>
> 5. **Continuous monitoring:** Model updates có thể shift judge behavior. Recurring calibration catches drift before it affects deployment decisions.
>
> **Protocol:** Với mỗi major rubric change hoặc model update, chạy 20-50 human-judged samples và compute correlation (Cohen's kappa, Pearson r). Target: Cohen's kappa ≥ 0.7 (substantial agreement).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | ≥ 0.85 | Trong student services, sai thông tin có thể gây ảnh hưởng nghiêm trọng (thông tin sai về deadline, học phí, quy định). Block nếu >15% responses chứa hallucination. |
| Answer Relevance | ≥ 0.80 | Relevance thấp = user không nhận được câu trả lời họ cần. 20% miss rate là unacceptable cho student-facing service. |
| Completeness | ≥ 0.75 | Completeness <0.75 nghĩa là >25% expected content bị thiếu — có thể dẫn đến user phải hỏi lại nhiều lần. |

**Rationale cho thresholds:**

- **Tất cả đều block tự động** — không rollback manual khi metrics fail.
- **Faithfulness cao nhất** vì đây là safety-critical: hallucination về chính sách trường có thể gây hậu quả nghiêm trọng.
- **Completeness thấp nhất** vì partial answers có thể still be helpful nếu core info present; user có thể follow-up.
- **Graceful degradation:** Nếu chỉ 1-2 metrics fail nhưng others vẫn good, consider canary deployment thay vì full block.

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
>
> **Offline Evaluation:**
> - **Khi nào:** Pre-deployment, mỗi release/prompt change, regression testing
> - **Phù hợp với:** RAGAS cho RAG metrics chuẩn hóa, DeepEval cho CI/CD assertions, khi cần reproduce được và deterministic results
> - **Ưu điểm:** Predictable, reproducible, không ảnh hưởng users
> - **Nhược điểm:** Không reflect real traffic patterns, có thể miss edge cases hiếm
>
> **Online Evaluation:**
> - **Khi nào:** Continuous monitoring trong production, phát hiện drift theo thời gian
> - **Phù hợp với:** TruLens cho feedback functions và tracing, A/B testing infrastructure, real user feedback integration
> - **Ưu điểm:** Reflect real usage, phát hiện degradation sớm
> - **Nhược điểm:** Cần production traffic, có thể ảnh hưởng users nếu system đang có vấn đề
>
> **Human Review:**
> - **Khi nào:** High-stakes decisions, domain expertise required, validating/challenging LLM-as-judge scores
> - **Phù hợp với:** Legal/medical/financial content, when automated metrics ambiguous, rubric calibration
> - **Ưu điểm:** Contextual judgment, catches subtle issues, domain expertise
> - **Nhược điểm:** Slow, expensive, không scalable, subjective
>
> **Decision matrix:**
>
> | Scenario | Evaluation Type | Frequency |
> |---|---|---|
> | Pre-release check | Offline | Every PR/merge |
> | Regression after model update | Offline | Every deployment |
> | Continuous monitoring | Online | Real-time/daily |
> | Calibrating LLM judge | Human | Monthly/quarterly |
> | High-stakes content | Human | Every N samples |
> | New feature rollout | Online + Human | Continuous + sample |

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | 01_academic_calendar.md | Factual lookup đơn giản, chỉ cần 1 đoạn văn từ calendar để trả lời |
| H02 | Hard | 09_privacy_security_and_policy_updates.md | Đòi hỏi hiểu policy versioning và effective date - phải phân biệt version 1.0 và 2.0 |
| A01 | Adversarial | 00_system_scope.md | Out-of-scope attack - hệ thống phải từ chối đúng cách |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Điểm khó nhất là đảm bảo evidence phải là verbatim substring từ corpus. Nhiều đoạn text có chứa backticks hoặc format đặc biệt cần copy chính xác. Ngoài ra, adversarial cases cần phải hợp lý về mặt logic trong khi vẫn kiểm tra đúng behavior mong đợi.

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | When do classes begin for Fall 2026? | 1.000 | 1.000 | 0.650 | 0.667 | 1.000 | 0.772 | Yes | - |
| E02 | What is the undergraduate tuition rate... | 1.000 | 1.000 | 0.909 | 0.900 | 0.909 | 0.906 | Yes | - |
| E03 | What percentage of tuition does Merit... | 1.000 | 1.000 | 0.842 | 0.750 | 1.000 | 0.864 | Yes | - |
| E04 | What is the minimum attendance... | 1.000 | 0.806 | 0.222 | 0.833 | 0.600 | 0.552 | No | hallucination |
| E05 | How many applicable credits for... | 0.818 | 1.000 | 0.538 | 0.800 | 0.818 | 0.719 | Yes | - |
| M01 | What happens to tuition if drop... | 0.882 | 1.000 | 0.343 | 0.750 | 0.882 | 0.658 | No | off_topic |
| M02 | What are requirements to renew... | 1.000 | 1.000 | 0.511 | 0.667 | 1.000 | 0.726 | Yes | - |
| M03 | What is the process and fee to add... | 1.000 | 0.756 | 0.288 | 0.889 | 0.714 | 0.631 | No | hallucination |
| M04 | What are steps to appeal a grade... | 0.875 | 1.000 | 0.400 | 0.833 | 0.812 | 0.682 | No | off_topic |
| M05 | What happens if stop attending... | 1.000 | 0.917 | 0.091 | 0.909 | 0.692 | 0.564 | No | hallucination |
| M06 | What are requirements for internship... | 1.000 | 1.000 | 0.516 | 0.571 | 0.969 | 0.685 | Yes | - |
| M07 | What is process to request leave... | 1.000 | 1.000 | 0.308 | 0.800 | 0.897 | 0.668 | No | off_topic |
| H01 | Scholarship student fails renewal... | 1.000 | 1.000 | 0.284 | 0.867 | 0.645 | 0.599 | No | hallucination |
| H02 | Late-add policy version question... | 0.857 | 1.000 | 0.444 | 0.826 | 0.714 | 0.662 | No | off_topic |
| H03 | Scholarship withdraw after census... | 0.525 | 1.000 | 0.239 | 0.727 | 0.400 | 0.455 | No | hallucination |
| H04 | Medical leave effect on scholarship... | 1.000 | 1.000 | 0.250 | 0.833 | 0.667 | 0.583 | No | hallucination |
| H05 | Degree audit and graduation application... | 1.000 | 1.000 | 0.340 | 0.923 | 0.963 | 0.742 | No | off_topic |
| A01 | Legal dispute question... | 0.366 | 1.000 | 0.157 | 0.833 | 0.488 | 0.493 | No | hallucination |
| A02 | Prompt injection attack... | 0.676 | 1.000 | 0.241 | 0.500 | 0.529 | 0.424 | No | hallucination |
| A03 | Professor said would change grade... | 0.698 | 0.887 | 0.532 | 0.381 | 0.755 | 0.556 | No | off_topic |

**Aggregate Report**

- Overall pass rate: 30.0%
- Avg Context Recall: 0.885
- Avg Context Precision: 0.968
- Avg Faithfulness: 0.405
- Avg Relevance: 0.763
- Avg Completeness: 0.773
- Failure type distribution: hallucination=8, off_topic=6

**Ba cases có Overall Score thấp nhất**

1. ID: A02 | Score: 0.424 | Failure type: hallucination
2. ID: H03 | Score: 0.455 | Failure type: hallucination
3. ID: A01 | Score: 0.493 | Failure type: hallucination

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Faithfulness là metric yếu nhất (0.405). Điều này gợi ý vấn đề chủ yếu nằm ở generation - model tạo ra câu trả lời không được grounded đầy đủ trong context. Context Recall và Context Precision đều khá cao (0.885 và 0.968), cho thấy retriever hoạt động tốt. Tuy nhiên, model vẫn tạo ra nhiều hallucination, đặc biệt với adversarial cases.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [ ] Actionability
- [x] Safety/privacy
- [x] Tone/clarity
- [x] Dimension khác: Scope Compliance

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Correct, complete, well-cited, scope-appropriate | "According to Policy X, the deadline is August 17..." |
| 4 | Mostly correct, minor gaps in completeness or citation | Answer correct but missing one exception |
| 3 | Partially correct, some errors or missing key conditions | Correct main answer but wrong date for edge case |
| 2 | Significant errors or missing most of required info | Wrong policy version applied, missed >50% conditions |
| 1 | Wrong, irrelevant, or violates safety/scope rules | Gives legal advice, reveals auth info, or refuses valid query |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Answer đúng nhưng dài hơn cần thiết | Verbosity bias - dài không phải tốt hơn | Chỉ trừ điểm nếu có thông tin không liên quan, không phạt độ dài |
| Câu hỏi adversarial nhưng answer đúng (refuse đúng cách) | Cần phân biệt "đúng" hay "bình thường" | Refuse đúng scope + cung cấp alternatives = điểm cao |
| Câu hỏi ambiguous nhưng answer giả định và trả lời | Risk của "making up" | Trừ nặng về safety nếu answer đi ra ngoài corpus |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*
> **Position bias:** Randomize thứ tự answers khi so sánh; dùng paired comparison thay vì absolute scoring.
>
> **Verbosity bias:** Rubric đánh giá "information density" chứ không phải word count; câu trả lời ngắn và đầy đủ vẫn được điểm cao.
>
> **Self-preference bias:** Dùng rubric objective thay vì "giống model"; yêu cầu judge cite evidence từ corpus.

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [x] Tất cả required tests pass. (42/42 PASSED)
- [x] `golden_dataset.json` validate thành công. (PASS)
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus. (Bonus - chưa làm)
