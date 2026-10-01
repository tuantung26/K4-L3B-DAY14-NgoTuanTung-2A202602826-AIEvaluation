# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 20.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.799 | 0.423 | 1.000 | Tốt. Retriever tìm thấy hầu hết nội dung cần thiết. |
| Context Precision | 0.922 | 0.583 | 1.000 | Rất tốt. Hầu hết chunk số 1 đều chứa thông tin hữu ích. |
| Faithfulness | 0.622 | 0.033 | 0.963 | Yếu. Có hiện tượng LLM bịa thông tin hoặc nói sai. |
| Relevance | 0.430 | 0.000 | 0.875 | Quá kém. Nhiều câu LLM lạc đề hoặc diễn đạt quá cộc lốc làm từ vựng không khớp. |
| Completeness | 0.674 | 0.111 | 1.000 | Chấp nhận được nhưng thiếu các bối cảnh điều kiện (ngoại lệ). |
| Overall Score | 0.609 | 0.201 | 0.819 | Nhìn chung hệ thống chưa đủ khả năng đưa vào sản xuất. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision, Context Recall.
- Metrics/cases ở mức Needs Work (0.6–0.8): Faithfulness, Completeness.
- Metrics/cases ở mức Significant Issues (<0.6): Relevance.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5.0% |
| irrelevant | 6 | 30.0% |
| incomplete | 0 | 0.0% |
| off_topic | 9 | 45.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở generation (do dùng LLM quá nhỏ `Gemini-Flash-Lite` và prompt chưa chặt chẽ). Cụ thể, các chỉ số Retrieval (Context Precision 0.922 và Context Recall 0.799) cho thấy BM25 làm tốt nhiệm vụ đưa đúng đoạn văn bản cho LLM. Tuy nhiên, Relevance (0.430) và Faithfulness (0.622) chỉ ra rằng generator không tuân thủ prompt, dẫn đến việc lấy thông tin không chính xác từ context, trả lời quá cộc lốc hoặc thiếu bối cảnh giải thích.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *H02:* My PulsePhone X screen has non-impact-related lines. If OrbitTech replaces the screen, how long is the new screen covered by warranty?

**Expected answer:**

> *Điền:* Since it is a covered defect, the replacement screen will be covered for the longer of 90 calendar days or the remainder of your original 24-month limited hardware warranty.

**Actual answer:**

> *Điền:* Based on the provided contexts, there is no mention of a separate or extended warranty period specifically for a newly replaced screen. The PulsePhone X itself comes with a 24-month limited hardware warranty...

**Scores:** Context Recall: 0.444 | Context Precision: 0.583 | Faithfulness: 0.033 | Relevance: 0.235 | Completeness: 0.333 | Overall: 0.201

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy thiếu đoạn policy chứa "Replacement parts are covered for the longer of 90 calendar days...". Vì thiếu evidence này, LLM đã bịa ra câu trả lời "there is no mention" gây ra hallucination.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | LLM kết luận sai vì không thấy thông tin bảo hành thay thế linh kiện. |
| Why 1 | Tại sao symptom xảy ra? | Vì thông tin bảo hành thay thế không nằm trong top-5 retrieved chunks. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Chunk đó không chứa đủ word overlap với câu hỏi (chứa "replacement parts" chứ không phải "screen"). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | BM25 chỉ dựa trên từ khóa khớp chính xác (lexical). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống RAG hiện tại không có công cụ tìm kiếm ngữ nghĩa (Semantic/Vector Search). |
| Why 5 | Root cause có thể hành động được là gì? | Cần nâng cấp Retrieval bằng cách dùng Embeddings hoặc Reranker. |

**Root cause từ `find_root_cause()`:**

> *Paste output:* Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Tôi đồng ý hoàn toàn. Metric Context Recall chỉ có 0.444 và Context Precision thấp (0.583), cho thấy retriever mang về các chunk rác và thiếu chunk quan trọng chứa đoạn 90-day parts warranty.

**Proposed fix cụ thể:**

> *Câu trả lời:* Thay thế hoặc bổ sung BM25 bằng Vector Search (Embeddings) để retriever hiểu được "screen replacement" có chung ngữ nghĩa với "replacement parts".

### Failure 2

**ID và question:**

> *E05:* How long is the limited warranty for the HomeHub Mini?

**Expected answer:**

> *Điền:* OrbitTech provides a 24-month limited hardware warranty for the HomeHub Mini.

**Actual answer:**

> *Điền:* 24 months

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.500 | Relevance: 0.000 | Completeness: 0.111 | Overall: 0.204

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy chuẩn xác đoạn văn chứa "24-month limited hardware warranty". (Recall/Precision đạt tuyệt đối). Nhưng LLM trả lời quá cộc lốc, không thành câu, làm metric Relevance (đo giao nhau từ vựng với câu hỏi) rơi xuống 0.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | LLM trả lời quá ngắn (cụt lủn), mất bối cảnh. |
| Why 1 | Tại sao symptom xảy ra? | System Prompt yêu cầu "Answer concisely". |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | LLM (Flash Lite) hiểu lầm "concisely" thành trả lời ngắn nhất có thể. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt không có rule bắt buộc trả lời trọn vẹn (complete sentences). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu Few-shot examples trong prompt. |
| Why 5 | Root cause có thể hành động được là gì? | Bổ sung hướng dẫn định dạng đầu ra (format instructions) vào system prompt. |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause (`find_root_cause()`): "Answer does not address the question — improve prompt clarity". Tôi đồng ý, fix là cập nhật system prompt: "Answer concisely but always respond in complete sentences".

### Failure 3

**ID và question:**

> *H01:* I placed an order for a new phone on August 15, 2026, and I want to return it unopened. How many days do I have to return it?

**Expected answer:**

> *Điền:* Since you placed the order before September 1, 2026, the Return Policy version 1.0 applies. You have 21 calendar days from confirmed delivery to return the unopened device.

**Actual answer:**

> *Điền:* You have 21 calendar days to return it.

**Scores:** Context Recall: 0.857 | Context Precision: 0.867 | Faithfulness: 0.667 | Relevance: 0.188 | Completeness: 0.286 | Overall: 0.380

**Evidence inspection:**

> *Câu trả lời:* LLM lấy đúng được đáp án "21 days" nhưng thiếu mất phần giải thích logic về mốc thời gian policy Version 1.0. Dẫn tới Completeness chỉ đạt 0.286.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | LLM không giải thích bối cảnh và điều kiện áp dụng. |
| Why 1 | Tại sao symptom xảy ra? | LLM chỉ chăm chăm trả lời kết quả cuối cùng (21 ngày). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt thiếu yêu cầu giải thích bối cảnh điều kiện (Chain of Thought). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Vì LLM nhỏ nên khả năng reasoning kém nếu không được ép "think step by step". |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có evaluation check cho phần giải thích. |
| Why 5 | Root cause có thể hành động được là gì? | Cần ép LLM thực hiện Chain-of-Thought (vd: trích dẫn điều kiện trước khi kết luận). |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause (`find_root_cause()`): "Answer does not address the question — improve prompt clarity". Fix: Yêu cầu LLM "If a policy has date conditions, explicitly state which condition applies before answering."

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Prompt quá lỏng lẻo làm LLM trả lời cộc lốc hoặc thiếu giải thích điều kiện. | E02, E05, H01, M04 | High |
| 2 | Semantic mismatch: BM25 miss các chunks đồng nghĩa (lexical gap). | H02, M01 | Medium |
| 3 | LLM quá nhỏ để tổng hợp thông tin phức tạp từ 2 văn bản trở lên. | M06, M07 | Low |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn Cluster 1. Chi phí rẻ nhất, tác động lớn nhất (chỉ cần đổi vài dòng trong prompt là có thể fix hàng loạt lỗi do LLM trả lời cộc lốc), từ đó kéo mạnh điểm Relevance và Completeness lên ngay lập tức.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Refine prompt to strictly address the user's question | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F004 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F005 | off_topic | Answer does not address the question — improve prompt clarity | Review full pipeline for issues | Open |
| F011 | hallucination | Context is missing or irrelevant — improve retrieval | Review full pipeline for issues | Open |
```

**Ba improvement suggestions ưu tiên**

1. Bổ sung Few-shot examples để ép output format chuẩn (trọn vẹn câu).
2. Thêm yêu cầu Chain-of-Thought giải thích điều kiện trước khi chốt đáp án.
3. Chuyển đổi Retriever sang mô hình Semantic Search hoặc Hybrid Search.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Bổ sung Few-shot examples định dạng trả lời tròn câu | Completeness, Relevance | Chạy lại `evaluate_answers.py` sau khi update prompt, xem Relevance của E05 có tăng không. |
| Bắt buộc Chain-of-Thought để giải thích rule | Completeness, Faithfulness | Benchmark lại, quan sát H01 xem có giải thích version 1.0 không. |
| Chuyển sang Semantic Search (Embeddings) | Context Recall | Chạy lại core evaluator để xem Recall của H02 có tăng không. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy trên mỗi Pull Request (PR) thay đổi logic retrieval, sửa đổi system prompt, cập nhật phiên bản model LLM mới, hoặc mỗi khi bộ tài liệu gốc (corpus) có đợt update lớn.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* Ngưỡng drop 0.05 khá nhạy và phù hợp ở giai đoạn này. Word overlap thường sẽ biến động nhẹ khi LLM đổi cách hành văn. Nếu drop > 0.05 chứng tỏ ít nhất 1-2 câu trong tập validation đã trả lời sai bối cảnh hoặc sinh ra ảo giác nặng (hallucination).

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:* Failures liên quan đến `hallucination` (bịa chính sách sai) hoặc tụt giảm mạnh ở metric `Faithfulness` và `Context Precision` phải **block deployment** ngay. Các lỗi như `Completeness` giảm nhẹ hoặc `irrelevant` (trả lời lòng vòng) chỉ nên **alert** để điều chỉnh sau.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Run Benchmark (Dev)] → [Review Metrics vs Baseline] → [Run Regression / LLMJudge] → Deploy
```

> *Giải thích:* Developer chạy benchmark offline lúc code, review xem metric có đạt yêu cầu tối thiểu không, rồi hệ thống CI/CD sẽ chạy Regression test trên tập data chính thức trước khi deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Refine Prompt (thêm Few-shot) | Completeness, Relevance | Khắc phục hàng loạt lỗi lạc đề/trả lời cộc lốc (như E05). |
| 2 | Upgrade BM25 thành Hybrid Search | Context Recall | Xử lý được các lỗi do miss lexical match (như H02). |
| 3 | Tích hợp LLM-as-a-Judge vào CI pipeline | Evaluation Reliability | Chấm điểm sát thực tế và ngữ nghĩa hơn thay vì phụ thuộc word overlap. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:* Nên thêm các câu hỏi phức tạp hơn về kết hợp bảo hành và giảm giá membership. Nên bổ sung vài câu hỏi user cố tình trích dẫn sai số liệu (để xem LLM có bị lừa không).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Metric Relevance (đo bằng word overlap) rơi tự do về 0% (ở case E05) mặc dù LLM đưa ra đúng câu trả lời (24 months). Tương tự, nếu LLM diễn đạt thông tin đúng nhưng paraphrase quá nhiều, hệ thống chấm điểm hiện tại của RAGAS bản rút gọn sẽ phạt rất khắt khe.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* Word-overlap không hiểu được nghĩa tương đồng (synonyms), không phân biệt được ý nghĩa ngữ cảnh và đặc biệt yếu khi áp dụng cho các ngôn ngữ linh hoạt. Trong production, bắt buộc phải dùng **LLM-as-a-Judge** (như GPT-4) kết hợp với mô hình Embeddings để đo **Semantic Similarity** (độ tương đồng ngữ nghĩa) thay cho Jaccard similarity.
