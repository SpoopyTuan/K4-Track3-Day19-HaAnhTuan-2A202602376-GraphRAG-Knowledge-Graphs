# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Hà Anh Tuấn  **MSSV:** 2A202602376  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

```
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small
top_k=3 | chunk_size=800 | chunks=176 | KG: 203 nodes / 382 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     57.6
graph       196     91958     4890   0.00944    134.2

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.80
graph       0.83   1.50     4787       86   0.00076     2.52
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00944 | ×8.43 |
| Indexing giây | 57.6 | 134.2 | ×2.33 |
| Mỗi câu: USD | 0.00013 | 0.00076 | ×5.85 |
| Mỗi câu: giây | 1.80 | 2.52 | ×1.40 |
| Mỗi câu: in_tok | 694 | 4787 | ×6.90 |

**Chi phí tăng thêm đến từ đâu?**
> GraphRAG gọi thêm LLM để trích xuất tin và ghi graph khi indexing. Khi trả lời, dữ kiện graph làm input tăng từ 694 lên 4787 token/câu, nên chi phí và độ trễ tăng. Các tỷ lệ tính từ số đã làm tròn trong file; chi phí LLM chấm `judge` được script theo dõi riêng, không nằm trong bảng chi phí pipeline.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai lấy đúng định nghĩa tiền chất. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai nêu đúng Trần Thanh Tuấn và Trần Minh Tâm. |
| Q3 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph | Graph nối được Điều 251 và khung cơ bản nhưng ghi sai 24 tháng thay vì 36 tháng. |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph | Graph xác định được hành vi và Điều 255 nhưng trả sai mức tối đa. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Graph nêu rõ Điều 250 khoản 4 và đúng khung hình phạt, còn Flat ghi “khoản b)”. |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 1 | Graph theo recall; hòa judge | Graph nêu rõ hơn tên người nhưng vẫn thiếu vụ Pháp y tâm thần và liệt kê hai mục có khả năng trùng vụ Cái Quang Huy. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Q4).

- **Hiện tượng:** GraphRAG trả mức tối đa 07 năm, không phải khung cao nhất của Điều 255 trong corpus.
- **Bằng chứng:** `ket_qua_benchmark_kg.txt`, Q4 Graph: “Hành vi này có thể bị phạt tù tối đa 07 năm theo Điều 255 BLHS.” Trong `data/drug_law/blhs-dieu-255.md`, khoản 4 ghi “thì bị phạt tù 20 năm hoặc tù chung thân”. Q4 Graph đạt recall 0.67, judge 1.
- **Nguyên nhân:** KG-3 trong `src/graph.py` chỉ lấy khoản 1 và khoản nhắc chất liên quan. Khoản 4 Điều 255 không nhắc tên chất, nên quy tắc này có thể bỏ sót khoản cần cho câu hỏi về mức tối đa.
- **Đề xuất sửa:** Khi câu hỏi yêu cầu mức tối đa, lấy toàn bộ khoản của Điều đã xác định thay vì lọc theo chất; tăng token nhưng tránh thiếu khung cao nhất. Đây là so sánh với corpus của bài, không suy ra mức án cụ thể của người bị bắt.

### Lỗi E4: Phép đo từ khóa chưa phản ánh đầy đủ nội dung (Q6).

- **Hiện tượng:** Flat có recall 0.00 nhưng judge 1 dù câu trả lời có thông tin liên quan.
- **Bằng chứng:** Q6 Flat nêu “Vụ việc của Thành” có 5 viên MDMA và “Vụ việc của Đông” có 0,686g MDMA. `data/benchmark_kg.json` yêu cầu các chuỗi “Cái Quang Huy”, “Lê Minh Thành”, “Pháp y tâm thần”; cách gọi tên ngắn không khớp các chuỗi này.
- **Nguyên nhân:** `must_include` đo sự xuất hiện của từ khóa, không giải quyết tên gọi tương đương. Judge chấm đúng một phần; tuy nhiên câu trả lời vẫn thiếu định danh vụ rõ ràng, nên không thể coi là đúng đầy đủ.
- **Đề xuất sửa:** Bổ sung nhóm tên tương đương cho phép đo sau khi đối chiếu nguồn, kết hợp đọc thủ công và judge; cần tránh chấp nhận tên ngắn mơ hồ như “Thành” nếu chưa xác định đúng người.

## 4. Kết luận (5 điểm)

> Flat RAG đủ cho câu hỏi lấy thông tin từ một nguồn như Q1–Q2: cả hai pipeline đều đạt recall 1.00, judge 2. GraphRAG phù hợp khi cần nối tin với luật: Q3–Q5 đều cải thiện điểm, recall trung bình tăng từ 0.43 lên 0.83 và judge từ 1.00 lên 1.50. Đánh đổi là indexing đắt hơn 8.43 lần, chi phí/câu cao hơn 5.85 lần và độ trễ tăng từ 1.80 lên 2.52 giây; vẫn cần kiểm tra lỗi mức án, lọc khoản và tổng hợp vụ.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.12s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 292 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 17 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00076. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy.

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Benchmark đã hoàn tất, chưa ghi nhận lỗi chạy lệnh còn tồn đọng. Lỗi chất lượng chưa sửa: Q3 trả 24 tháng thay vì 36 tháng; Q4 bỏ sót mức tối đa; Q6 thiếu vụ Pháp y tâm thần và có hai mục có khả năng trùng vụ Cái Quang Huy. Chưa có bằng chứng truy vấn graph để xác định nguyên nhân Q3 và xác nhận trùng thực thể ở Q6.
