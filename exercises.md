# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Khi trả lời câu hỏi mở, creative hoặc tóm tắt ý chính | Sai lệch thông số kỹ thuật, giá cả, hoặc thông tin bảo hành | Tinh chỉnh prompt, thêm guardrails ép LLM bám sát context |
| Answer Relevance | Khi câu hỏi tối nghĩa và agent đặt câu hỏi làm rõ | Agent trả lời lạc đề hoàn toàn hoặc lặp lại thông tin không liên quan | Cải thiện query understanding, sửa prompt focus |
| Context Recall | Trả lời đúng nhờ internal knowledge của LLM dù context thiếu | Context thiếu thông tin trọng yếu khiến LLM trả lời sai (hallucinate) | Chỉnh sửa chunking strategy, cải thiện thuật toán retrieval |
| Context Precision | Chunks đúng bị xếp hạng thấp nhưng vẫn nằm trong context window | Chunks đúng bị đẩy ra khỏi giới hạn context window gây mất thông tin | Thêm hoặc cải thiện mô hình Reranking |
| Completeness | Người dùng yêu cầu tóm tắt ngắn gọn | Người dùng yêu cầu danh sách chi tiết nhưng agent bỏ sót nhiều mục | Tăng số lượng retrieved chunks (top-k), sửa prompt |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Thiết kế A/B testing với cùng một prompt: Condition 1 đưa Answer A lên trước Answer B, Condition 2 đưa Answer B lên trước Answer A. Đánh giá xem LLM Judge có xu hướng luôn chọn câu trả lời ở vị trí đầu tiên bất kể nội dung hay không.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Định nghĩa rõ ràng trong rubric rằng "Độ dài không phản ánh chất lượng". Yêu cầu LLM Judge phạt điểm những câu trả lời dài dòng, lan man và thưởng điểm cho những câu trả lời súc tích, đi thẳng vào trọng tâm vấn đề.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM Judge có thể có những bias riêng (quá khắt khe hoặc quá nới lỏng) và hiểu sai tiêu chí domain-specific. So sánh và căn chỉnh kết quả của LLM Judge với đánh giá của con người giúp đảm bảo độ tin cậy và phản ánh đúng giá trị thực tế mong muốn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Thông tin support (kỹ thuật, chính sách) phải cực kỳ chính xác, tránh hallucination gây thiệt hại. |
| Answer Relevance | 0.70 | Cần trả lời đúng trọng tâm nhưng có thể linh hoạt khi người dùng hỏi các câu hỏi mở. |
| Completeness | 0.75 | Đảm bảo cung cấp đủ các bước hướng dẫn hoặc thông tin cần thiết để giải quyết vấn đề của user. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* 
> - **Offline evaluation:** Dùng trong quá trình dev/test/CI để kiểm thử prompt/model mới trên tập golden dataset trước khi deploy.
> - **Online evaluation:** Dùng trên production (dựa vào user feedback, implicit signals như click, time on page) để đo lường performance thực tế.
> - **Human review:** Dùng để tạo golden dataset ban đầu, kiểm tra các edge cases phức tạp, hoặc khi auto-metrics có dấu hiệu mâu thuẫn/bất thường.

---

## Part 2 — Core Coding (9:45–10:40)

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
| 5 | Hoàn hảo: Chính xác 100% kỹ thuật/chính sách, súc tích, thái độ chuyên nghiệp, giải quyết triệt để vấn đề. | "Sản phẩm A được bảo hành 12 tháng. Để kích hoạt, bạn vui lòng truy cập link sau: [link]." |
| 4 | Tốt: Trả lời chính xác, giải quyết được vấn đề nhưng có thể hơi dài dòng hoặc thiếu một chi tiết nhỏ không quá quan trọng. | "Sản phẩm A có bảo hành 12 tháng theo chính sách của công ty. Bạn có thể kích hoạt qua website. Nếu cần thêm hỗ trợ hãy báo tôi." |
| 3 | Chấp nhận được: Thông tin cơ bản đúng nhưng cách diễn đạt khó hiểu, thiếu bước hướng dẫn rõ ràng. | "Có bảo hành 12 tháng nha bạn, tự lên web công ty mà kích hoạt bảo hành." |
| 2 | Kém: Thiếu nhiều thông tin quan trọng hoặc có sai sót nhỏ về kỹ thuật/giá cả gây hiểu lầm. | "Sản phẩm A bảo hành 24 tháng (sai thông tin)." |
| 1 | Tệ hại: Cung cấp sai hoàn toàn thông tin quan trọng, từ chối hỗ trợ sai cách, hoặc thái độ thô lỗ. | "Tôi không biết, bạn tự tìm hiểu đi." |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Câu hỏi user mơ hồ | Không có ground truth rõ ràng để đối chiếu tính Completeness. | Thưởng điểm 5 nếu agent biết đặt câu hỏi làm rõ (clarifying questions) lịch sự. |
| Đúng thông tin nhưng quá nhiều thuật ngữ | Technically correct nhưng user experience kém do khó hiểu. | Đưa tiêu chí "Tone/clarity - Dễ hiểu với người dùng phổ thông" vào rubric để giới hạn ở mức 3 hoặc 4. |
| Hỏi ngoài lề (Out of scope) | LLM có thể trả lời đúng câu hỏi ngoài lề nhưng lại vi phạm rule của hệ thống. | Quy định rõ: Trả lời từ chối khéo léo và hướng về sản phẩm OrbitTech sẽ được điểm tối đa (5). |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias, verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* 
> - **Position bias:** Tráo đổi vị trí các đáp án trong prompt khi so sánh (Swap test).
> - **Verbosity bias:** Ghi rõ trong rubric tiêu chí "súc tích" và phạt điểm các câu trả lời dài dòng không mang lại giá trị thêm.
> - **Self-preference:** Cung cấp few-shot examples đa dạng về văn phong để LLM không chỉ ưu tiên văn phong giống chính nó.

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

> *Câu trả lời:* Vì Context Recall đo lường mức độ bao phủ của expected answer trong TOÀN BỘ tập retrieved chunks (thường tính bằng set intersection hoặc LLM extract trên tổng thể). Reranking chỉ thay đổi thứ tự (order) của các chunks chứ không thay đổi tập hợp các chunks, nên lượng thông tin tổng thể cung cấp cho LLM không thay đổi.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:* Khi Context Recall thấp (thông tin cần thiết thực sự không tồn tại trong tập retrieved chunks). Lúc này, reranking không có tác dụng vì "không có bột mới gột nên hồ". Ta cần phải tinh chỉnh thuật toán retriever (dùng hybrid search), cải thiện query formulation (query expansion), hoặc xem lại chiến lược chunking để đảm bảo thông tin không bị cắt nát.

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
