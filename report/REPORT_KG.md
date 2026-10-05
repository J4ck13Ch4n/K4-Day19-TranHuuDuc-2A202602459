# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Hữu Đức  **MSSV:** 2A202602459  **Ngày:** 2026-10-05

> Provider: `custom:gpt-4o-mini` (OpenAI-compatible gateway `CUSTOM_BASE_URL`) + embedding `custom:text-embedding-3-small`, top_k=3, chunk_size=800, chunks=176, KG: 204 nodes / 381 rels (file benchmark). Graph live khi chụp ảnh có thể lệch ±1–2 node/cạnh do LLM trích xuất không ổn định giữa các lần chạy (`LAB_GUIDE.md` mục Q-A: “mỗi lần chạy lệch một chút”) — lần `--build` cuối: 205 node / 382 cạnh (thêm 1 `Person` + 1 `CHARGED_WITH`). Giá USD ước tính theo bảng trong `src/llm.py` (gpt-4o-mini 0.15/0.60 USD/1M tok). Neo4j 5.26.0 community chạy local (tarball, do tag `neo4j:5` trên Docker Hub bị kẹt layer). Hỗ trợ provider `custom` được thêm vào `src/llm.py` (cộng thêm, không sửa logic test/bench).

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    128.0
graph       196     91958     4690   0.00932    217.8

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     3.01
graph       0.69   1.33     3242       80   0.00053     3.52
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00932 | ×8.3 |
| Indexing giây | 128.0 | 217.8 | ×1.7 |
| Indexing calls | 176 | 196 | ×1.11 (20 calls trích xuất tin) |
| Mỗi câu: USD | 0.00013 | 0.00053 | ×4.1 |
| Mỗi câu: giây | 3.01 | 3.52 | ×1.17 |
| Mỗi câu: in_tok | 694 | 3242 | ×4.7 |
| Mỗi câu: recall (mean) | 0.43 | 0.69 | +0.26 |
| Mỗi câu: judge (mean) | 1.00 | 1.33 | +0.33 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Indexing tăng ×8.3 USD vì 20 lần gọi LLM trích xuất tin (`extract_news_cases`, out_tok 4690) + embed 176 chunk vẫn giữ nguyên; thời gian chỉ ×1.7 vì phần lớn là embed. Mỗi câu hỏi tăng ×4.7 in_tok vì `context()` thêm ~20 dữ kiện graph (khoản luật + tóm tắt vụ) vào prompt; độ trễ chỉ +0.5s vì 1 hop Cypher rẻ. Điểm hòa vốn: indexing chênh +0.0082 USD, mỗi câu Graph đắt hơn ~0.0004 USD nhưng không rẻ hơn theo số câu (Graph luôn đắt hơn/câu); Graph chỉ “đáng” khi giá trị của +0.26 recall (đặc biệt Q3 cross-kb từ 0→1) vượt chi phí — tức hệ hỏi nhiều câu cross-kb/aggregation, không phải để tiết kiệm tiền.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Đáp án nằm gọn trong 1 chunk luật (tiền chất); graph không thêm gì. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên 2 bị cáo tử hình nằm gọn trong 1 bài báo; vector top-3 đủ. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat “Không đủ thông tin” vì án (tin) và Điều 251 + khung 02–07 năm (luật) nằm 2 KB; graph đi `Person→Case→Crime→Article→Clause`. |
| Q4 | cross-kb | 0.00 / 0 | 0.00 / 0 | Hòa (thua) | Cả hai “Không đủ thông tin” (xem E2): vụ Hoàng Nato có `INVOLVES etomidate/ketamine/thuốc lắc` nhưng luật Điều 255 `MENTIONS` chất khác, bộ lọc khoản hụt. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph | Vụ Cái Quang Huy cần tội + MDMA 9,6kg + khoản 4 + tử hình; Flat thiếu số Điều/khoản, Graph đủ khoản 4 + tử hình nhưng nhầm Điều 251 (phải 250) — xem E5. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph | Cần liệt kê nhiều vụ có MDMA; vector top-3 thiếu + bịa (“vụ của Đức/Đông”), graph liệt kê từ `INVOLVES MDMA` nên trúng “Pháp y tâm thần”. |

Quy luật: single-hop → hòa (vector đủ); cross-kb/aggregation → Graph thắng hoặc hòa-thua khi bộ lọc khoản / chuẩn hóa chất gãy (Q4).

## 3. Phân tích lỗi (20 điểm)

### Lỗi E3: Trùng thực thể — một chất thành nhiều node

- **Hiện tượng:** `Substance` có cặp chỉ khác chữ hoa và nhiều tên chung chung/slang cùng chỉ MDMA nhưng thành node riêng.
- **Bằng chứng:** Cypher và kết quả thật trên graph đầy đủ (204 node):

```cypher
MATCH (s:Substance) RETURN s.name ORDER BY toLower(s.name);
```

```
s.name
"Amphetamine"
"chất ma túy"
"Cocaine"
"côca"
"cần sa"
"etomidate"
"Heroine"
"Ketamine"
"ketamine"
"ma túy"
"ma túy tổng hợp"
"MDMA"
"Methamphetamine"
"methamphetamine"
"thuốc lắc"
"thuốc phiện"
"XLR-11"
```

`Ketamine` vs `ketamine`, `Methamphetamine` vs `methamphetamine` là cùng chất; `thuốc lắc` là slang của `MDMA`; `ma túy`/`chất ma túy`/`ma túy tổng hợp` là tên chung, không phải chất cụ thể.

- **Nguyên nhân:** nằm ở **thiết kế ontology + `link_entity` chỉ dùng cho `Crime`**: `Substance` được `MERGE theo name thô` (`add_news_case`, `add_law_article`), LLM trích tên tự do theo báo (“ketamine” thường vs “Ketamine” chuẩn trong `SUBSTANCES`), không qua chuẩn hóa/linking. `SUBSTANCES` trong code cũng đã chứa cả biến thể nhưng không có bước map.
- **Đề xuất sửa:** thêm `normalize_substance` (lowercase, map đồng nghĩa: `thuốc lắc/kẹo → MDMA`, `ma túy tổng hợp → MDMA|Methamphetamine?` — cần từ điển, hoặc loại tên chung ra khỏi `Substance` thành property) + dùng `link_entity(..., normalize=...)` cho chất trong `extract_news_cases` và `find_substances`, giống tội danh; `MERGE` theo tên chuẩn. Đánh đổi: cần từ điển tay, tốn công; gộp sai (VD `ma túy đá → Methamphetamine`) còn tệ hơn trùng.

### Lỗi E2: Thiếu ngữ cảnh luật — Q4 “Không đủ thông tin” dù graph có Điều 255

- **Hiện tượng:** Q4 (Hoàng Nato: hành vi gì? tối đa bao nhiêu?) cả Flat và Graph đều trả “Không đủ thông tin”, recall 0.00/0.00, dù graph có `Article Điều 255` và vụ Hoàng Nato.
- **Bằng chứng:** trích file kết quả:

```
--- Q4 [cross-kb] flat recall=0.00 judge=0
Không đủ thông tin.
--- Q4 [cross-kb] graph recall=0.00 judge=0
Không đủ thông tin.
```

Cypher:

```cypher
MATCH (k:Case)-[:INVOLVES]->(s:Substance) RETURN k.name, s.name LIMIT 30;
```

```
"Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy", "etomidate"
"Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy", "ketamine"
"Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy", "thuốc lắc"
```

Vụ có 3 chất, trong đó `thuốc lắc` (= MDMA slang) và `ketamine` thường không khớp tên chuẩn mà Điều 255 `MENTIONS` (Heroine/Cocaine/Methamphetamine/… viết hoa chuẩn), cộng E3 (trùng hoa/thường) làm `EXISTS { (k)-[:INVOLVES]->(s)<-[:MENTIONS]-(cl) }` trong `context()` hụt; khoản 4 (chung thân) bị lọc mất, chỉ còn khoản 1 nên LLM từ chối trả lời.

- **Nguyên nhân:** **Cypher lọc khoản ở KG-3** (chỉ giữ khoản 1 + khoản `MENTIONS` chất mà vụ `INVOLVES`) + **ontology không chuẩn hóa chất** (E3) + ngưỡng nằm trong text không query được. Không phải do thiếu node luật.
- **Đề xuất sửa:** (1) chuẩn hóa chất như E3; (2) nới KG-3: luôn lấy khoản 1 + khoản cao nhất của Điều đã định danh (hoặc mọi khoản khi `INVOLVES` rỗng) thay vì lọc chặt; (3) tách ngưỡng thành property `Clause.min_g/max_g` để suy luận Q5/Q4 chính xác. Đánh đổi: prompt dài hơn (~+1-2k tok/câu), tốn hơn nhưng hết “mù” khung nặng.

### (Bổ sung, không tính điểm) E5+E4: Q5 nhầm Điều, Q6 recall–judge lệch

- Q5 Graph: `...điều luật tương ứng được áp dụng là Điều 251 BLHS khoản 4...` — sai, gold là **Điều 250** (vận chuyển), 251 là mua bán. Nguyên nhân: 2 tội cùng `MENTIONS MDMA`, Cypher lấy mọi Article qua `Crime` của Case mà Case bị gán 2 tội hoặc `Crime` cầu nối lấy nhầm; LLM chọn Điều đầu tiên. Recall vẫn 0.80 vì chứa “vận chuyển/MDMA/khoản 4/tử hình”, judge=1 (đúng một phần) — phép đo `recall` máy móc không phạt sai số Điều.
- Q6 Flat bịa (“vụ của Đức/Đông”) nhưng judge=1, recall=0.00; Graph liệt kê 4 vụ nhưng thiếu tên “Cái Quang Huy”/“Lê Minh Thành” (chỉ mô tả “vụ vận chuyển từ Đức…”, “vụ góp tiền…”) nên recall chỉ 0.33 dù nội dung đúng hơn — `must_include` khớp chuỗi cứng, không hiểu đồng nghĩa.

## 4. Kết luận (5 điểm)

Nên dùng KG khi câu hỏi **xuyên 2 KB** (án ở tin + Điều/khung ở luật: Q3) hoặc **aggregation** (liệt kê nhiều vụ theo chất: Q6): Graph recall 1.00 vs 0.00 (Q3), 0.33 vs 0.00 (Q6), judge trung bình 1.33 vs 1.00. Flat RAG đủ khi đáp án nằm gọn 1 đoạn (Q1/Q2 hòa 1.00/2–2). Giá phải trả: indexing ×8.3 USD (+0.0082 USD một lần, chủ yếu 20 calls trích xuất), mỗi câu ×4.1 USD (+0.0004 USD) và ×4.7 in_tok, trễ +17%. Với hệ ít câu (<100) và toàn single-hop, Flat đủ; với hệ hỏi lặp lại nhiều câu cross-kb trên cùng graph (tòa soạn/pháp chế), chi phí indexing một lần được khấu hao và +0.26 recall đáng tiền — với điều kiện sửa E2/E3 (chuẩn hóa chất, nới lọc khoản), nếu không Q4-type vẫn trắng tay.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.23s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = custom:gpt-4o-mini | embedding = custom:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00078. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ vận chuyển ma túy từ Đức về Việt Nam → Điều 250, khoản 4 MDMA ≥100g; tránh trùng Lê Minh Thành trong hướng dẫn).

> Lưu ý chụp ảnh: Neo4j đang chạy local (community 5.26.0, `http://localhost:7474`, user `neo4j`/pass `password123`, xem `LAB_GUIDE.md` Bước 8.1). Chạy `:clear` trước mỗi ảnh. Q-A đếm node; Q-B cầu nối; Q-D thay `'Lê Minh Thành'` bằng `'Cái Quang Huy'`:
```cypher
MATCH p=(:Person {name:'Cái Quang Huy'})-[:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article)
OPTIONAL MATCH q=(k)-[:INVOLVES|LOCATED_IN]->()
RETURN p, q;
```

## Vấn đề gặp phải (không tính điểm)

1. Tag `neo4j:5` trên Docker Hub kẹt ở 4 layer `Pulling fs layer` 20+ phút không tiến triển (trong khi `postgres:16`, `hello-world`, `neo4j:4.4`, `neo4j:5.26.0` tải được). Chuyển sang `neo4j:5.26.0` cho Docker và cuối cùng chạy Neo4j community 5.26.0 bằng tarball + JDK 21 có sẵn (`~/tools/jdk-21.0.12.1+1`) tại `/tmp/opencode/neo4j-setup/neo4j-community-5.26.0` (`bin/neo4j start`, pass `password123`).
2. `OPENAI_API_KEY` trong `.env` báo 401 `token_invalidated` (đã lộ ở log). Người dùng cấp custom gateway (`CUSTOM_BASE_URL=https://api.shopaikey.com/v1`, `CUSTOM_API_KEY`, `LLM_PROVIDER=custom`, `LLM_MODEL=gpt-4o-mini`); đã thêm hỗ trợ provider `custom` (OpenAI-compatible) vào `src/llm.py` và `EMBEDDING_PROVIDER=custom`. Kiểm tra chat+embed OK trước khi benchmark.
3. Không có desktop browser kết nối nên chưa chụp 3 ảnh Neo4j Browser; các truy vấn Q-A/Q-B/Q-D đã kiểm tra bằng `cypher-shell` và cho kết quả tốt (7 label, 204 node / 381 cạnh, 78 đường cầu nối). Người dùng tự chụp theo mục 8.1–8.2.
