# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi mang tính chào hỏi, xã giao hoặc từ chối lịch sự ngoài phạm vi hỗ trợ (không phụ thuộc vào văn bản context). | Trợ lý bịa đặt chính sách đổi trả, thời hạn bảo hành, thông số kỹ thuật khác với tài liệu nguồn (hallucination). | Siết chặt system prompt yêu cầu chỉ dựa trên context, giảm temperature = 0, bổ sung citation/grounding check. |
| Answer Relevance | Câu hỏi của người dùng mơ hồ, thiếu thông tin buộc trợ lý phải phản hồi bằng câu hỏi làm rõ (clarifying question). | Người dùng hỏi quy trình hoàn tiền nhưng trợ lý trả lời thông tin giới thiệu công ty hoặc cấu hình sản phẩm không liên quan (off-topic). | Cải thiện prompt hướng dẫn bám sát trọng tâm câu hỏi; bổ sung bước phân loại ý định (intent routing) và query rewrite. |
| Context Recall | Câu hỏi đơn giản, định nghĩa ngắn chỉ cần trích xuất 1 chi tiết trong 1 chunk duy nhất mà không cần gom toàn bộ thông tin phụ. | Câu hỏi phức tạp đòi hỏi tổng hợp nhiều điều kiện (vd: chính sách bồi hoàn linh kiện) nhưng retriever bỏ sót chunk quan trọng. | Tăng top-k retrieval, tối ưu kích thước chunk và overlap, tích hợp hybrid search (Dense + BM25) hoặc query expansion. |
| Context Precision | Generator có khả năng kháng nhiễu tốt (robust) và chunk liên quan vẫn nằm trong top 3 kết quả truy xuất. | Chunk đúng rơi xuống cuối danh sách (vị trí 4-5) trong khi các chunk nhiễu/lạc đề chiếm đầu danh sách khiến LLM bị dẫn dắt sai. | Tích hợp reranker (cross-encoder hoặc lexical overlap reranking) để sắp xếp lại tài liệu, đẩy chunk liên quan lên đầu prompt. |
| Completeness | Câu hỏi mang tính thăm dò hoặc yêu cầu tóm tắt ngắn gọn, người dùng chỉ cần câu trả lời khái quát thay vì toàn bộ chi tiết. | Khách hàng hỏi thủ tục giấy tờ bảo hành hoặc các bước đổi hàng nhưng trợ lý chỉ liệt kê thiếu các điều kiện tiên quyết. | Áp dụng Chain-of-Thought (CoT) yêu cầu checklist đầy đủ các ý trong prompt sinh câu trả lời; thêm self-verification. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition 1 (Baseline - Order AB):** Cung cấp cặp câu trả lời cho LLM Judge theo thứ tự `[Answer A, Answer B]` và yêu cầu judge chấm điểm hoặc chọn câu trả lời tốt hơn cho cùng một câu hỏi và context.
> - **Condition 2 (Swapped - Order BA):** Hoán đổi thứ tự thành `[Answer B, Answer A]`, giữ nguyên prompt và tiêu chí chấm, cho cùng model judge chấm độc lập.
> - **Phân tích đo lường:** So sánh tỷ lệ thắng của vị trí thứ nhất (Position 1 Win Rate). Nếu tỷ lệ câu trả lời đứng ở vị trí 1 được chọn vượt trội (ví dụ > 60-65%) ở cả 2 condition bất kể nội dung là A hay B, chứng tỏ judge có position bias rõ rệt. Để giảm thiểu, pipeline đánh giá cần chạy cả 2 chiều và tính điểm trung bình (swap-and-average).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - **Chấm điểm theo Fact Checklist (Content Density):** Thiết kế rubric quy định rõ các tiêu chí thông tin bắt buộc phải có thay vì đánh giá cảm tính độ chi tiết; chỉ cộng điểm khi có fact đúng, không tính điểm cho sự dài dòng.
> - **Ràng buộc tiêu chí súc tích (Conciseness Penalty):** Trong rubric mức điểm 5, quy định rõ câu trả lời phải "ngắn gọn, trực diện, không chứa thông tin thừa/filler phrases". Nếu dài dòng lan man sẽ bị hạ xuống mức điểm thấp hơn.
> - **Chuẩn hóa đầu vào:** Quy định format trả lời có cấu trúc (ví dụ: bullet points, bảng ngắn) để hạn chế khoảng cách về độ dài giữa các câu trả lời trước khi đưa vào judge.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - **Đo lường mức độ đồng thuận (Alignment):** Sử dụng các chỉ số như Cohen's Kappa hoặc Pearson/Spearman correlation để kiểm tra xem đánh giá của LLM Judge có tương quan chặt chẽ với chuyên gia con người (human gold standard) hay không.
> - **Phát hiện và hiệu chỉnh Systematic Biases:** LLM Judge thường có thiên kiến tiềm ẩn (self-preference, leniency bias). Việc đối chiếu với human labels giúp phát hiện các điểm mù này để tinh chỉnh rubric và prompt của judge.
> - **Xác định ngưỡng tin cậy (Confidence Gate):** Giúp quyết định ngưỡng điểm nào là an toàn để hệ thống tự động hóa đánh giá và khoảng điểm nào (borderline) cần chuyển cho con người xem xét thủ công.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Trong hệ thống CSKH, Hallucination là rủi ro nghiêm trọng nhất có thể gây sai lệch chính sách bảo hành, cam kết sai cho khách hàng và dẫn tới rủi ro pháp lý/tài chính. |
| Answer Relevance | 0.75 | Đảm bảo trợ lý trả lời đúng trọng tâm nhu cầu của khách hàng, không trả lời vòng vo, lạc đề hoặc né tránh vấn đề gây bức xúc cho người dùng. |
| Completeness | 0.70 | Đảm bảo câu trả lời chứa đủ các bước hướng dẫn hoặc điều kiện cốt lõi để khách hàng tự giải quyết được vấn đề mà không phải hỏi đi hỏi lại nhiều lần. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Dùng trong giai đoạn phát triển (Development), kiểm thử trước khi release (Staging/Pre-production) và tích hợp vào CI/CD pipeline. Chạy tự động trên bộ Golden Dataset (20 QA) để phát hiện regression ngay khi có thay đổi về prompt, chunking, retriever hay model.
> - **Online Evaluation:** Dùng khi hệ thống đã deploy lên Production phục vụ người dùng thật. Đo lường liên tục qua telemetry thời gian thực: tỷ lệ phản hồi người dùng (thumbs up/down), tỷ lệ escalate sang nhân viên hỗ trợ, latency, token usage, và chạy LLM judge định kỳ trên log hội thoại thực tế.
> - **Human Review:** Dùng định kỳ (weekly/monthly audit) bằng cách lấy mẫu ngẫu nhiên (sampling) để calibrate LLM Judge, hoặc kích hoạt cho các ca biên giới (borderline scores gần threshold), các khiếu nại nghiêm trọng của khách hàng, hoặc khi mở rộng thêm miền tri thức mới.

---

## Part 2 — Core Coding (14:45–15:40)

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

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

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
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

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

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
