# Báo cáo Thực nghiệm — Ngày 19: Flat RAG vs GraphRAG

> **Họ tên:** Nguyễn Thanh Giang  **MSSV:** 2A202602576  **Ngày:** 2026-10-05
> **LLM / Embedding sử dụng:** `openai:gpt-4o-mini` / `openai:text-embedding-3-small`

---

## E1. Bảng kết quả thực nghiệm (1.0 đ)

*Số liệu trích xuất từ `ket_qua_benchmark_kg.txt` (chạy lệnh `python bench_kg.py --judge`). Đồ thị xây dựng được gồm **206 nút** (`Clause`: 99, `Person`: 37, `Article`: 18, `Substance`: 17, `Case`: 15, `Crime`: 13, `Location`: 7) và **388 cạnh**.*

### 1.1. Chi phí & Thời gian theo pha (Indexing vs Querying)

| Metric | Flat RAG | GraphRAG | Tỷ số (Graph / Flat) |
|---|---:|---:|---:|
| **Indexing — Số lần gọi API (`calls`)** | 176 | 196 | **1.11×** |
| **Indexing — Input tokens (`in_tok`)** | 56,072 | 91,958 | **1.64×** |
| **Indexing — Output tokens (`out_tok`)** | 0 | 4,993 | **∞** (4,993 vs 0) |
| **Indexing — Chi phí (`USD`)** | `\$0.00112` | `\$0.00950` | **8.48×** |
| **Indexing — Thời gian (`seconds`)** | 49.8s | 143.4s | **2.88×** |
| **Querying (TB/câu) — Fact Recall** | 0.43 | 0.83 | **1.93×** (+0.40) |
| **Querying (TB/câu) — LLM Judge (0–2)** | 1.00 | 1.67 | **1.67×** (+0.67) |
| **Querying (TB/câu) — Input tokens (`in_tok`)** | 694 | 4,791 | **6.90×** |
| **Querying (TB/câu) — Output tokens (`out_tok`)** | 47 | 87 | **1.85×** |
| **Querying (TB/câu) — Chi phí (`USD`)** | `\$0.00013` | `\$0.00076` | **5.85×** |
| **Querying (TB/câu) — Độ trễ (`seconds`)** | 1.56s | 5.73s | **3.67×** |

### 1.2. Kết quả chi tiết trên từng câu hỏi (Q1 – Q6)

| Câu hỏi | Loại (`category`) | Số bước nhảy (`hops`) | Flat Recall | Graph Recall | $\Delta$ Recall | Flat Judge | Graph Judge |
|---|---|---:|---:|---:|---:|---:|---:|
| **Q1** | `single-hop-law` | 1 | 1.00 | 1.00 | **0.00** | 2 | 2 |
| **Q2** | `single-hop-news` | 1 | 1.00 | 1.00 | **0.00** | 2 | 2 |
| **Q3** | `cross-kb` | 3 | 0.00 | 1.00 | **+1.00** | 0 | 2 |
| **Q4** | `cross-kb` | 3 | 0.00 | 0.67 | **+0.67** | 0 | 1 |
| **Q5** | `cross-kb-multi-hop` | 4 | 0.60 | 1.00 | **+0.40** | 1 | 2 |
| **Q6** | `aggregation` | $\ge 5$ | 0.00 | 0.33 | **+0.33** | 1 | 1 |
| **Trung bình** | — | — | **0.43** | **0.83** | **+0.40** | **1.00** | **1.67** |

---

## E2. Phân tích từng loại câu hỏi (1.5 đ)

### 1. Nhóm `single-hop` (`Q1`: `single-hop-law`, `Q2`: `single-hop-news`)

- **Kết quả:** Cả Flat RAG và GraphRAG đều đạt điểm tuyệt đối (`Recall = 1.00`, `Judge = 2/2`).
- **Giải thích cơ chế:**
  - Với **Q1** (*"Theo Điều 251 Bộ luật Hình sự, hành vi mua bán trái phép chất ma túy bị phạt tù tối thiểu bao nhiêu năm ở khoản 1?"*), toàn bộ các fact cần thiết (`"Điều 251"`, `"2 năm"`, `"9 năm"`) cùng nằm gọn trong **một chunk duy nhất** của văn bản luật (`data/law/blhs-2015-chuong-xx-ma-tuy.md`). Tương đồng cosine (dense vector) kết hợp BM25 của `HybridRetriever` (`k=5`) dễ dàng xếp chunk Điều 251 lên vị trí top-1 nhờ trùng khớp từ khóa `"Điều 251"` và `"mua bán trái phép chất ma túy"`.
  - Với **Q2** (*"Bị cáo Lê Minh Thành bị bắt giữ với bao nhiêu gam ma túy đá và tại địa phương nào?"*), cả 3 fact (`"Lê Minh Thành"`, `"914"`, `"TP HCM"`) cũng xuất hiện cục bộ ngay trong phần mở đầu của bài báo `news-100260918080821054.md`.
  - **Kết luận cho `single-hop`:** Khi toàn bộ thông tin cần trả lời nằm trong phạm vi 1 chunk đơn lẻ, tìm kiếm tương đồng ngữ nghĩa/từ khóa (Flat RAG) đã đủ đạt `Recall = 1.00` với chi phí truy vấn rẻ hơn **5.85 lần** và nhanh hơn **3.67 lần** so với GraphRAG. Việc đi qua đồ thị ở đây không mang lại lợi thế thêm về độ chính xác.

### 2. Nhóm `cross-kb` & `cross-kb-multi-hop` (`Q3`, `Q4`, `Q5`)

- **Kết quả:** Đây là nhóm thể hiện sự khác biệt một trời một vực giữa hai kiến trúc:
  - **Q3** (*"Bị cáo Nguyễn Văn Hùng bị truy tố về tội danh gì, tội danh đó được quy định tại Điều mấy của Bộ luật Hình sự, và khung hình phạt cao nhất của điều luật đó là gì?"*): Flat RAG thất bại hoàn toàn (`Recall = 0.00`, `Judge = 0`), trong khi GraphRAG đạt tuyệt đối (`Recall = 1.00`, `Judge = 2`).
  - **Q4** (*"Trong vụ án của bị cáo Trần Hoàng Sơn, tang vật thu giữ là chất ma túy gì, khối lượng bao nhiêu, và theo Bộ luật Hình sự thì khối lượng đó thuộc khoản mấy của điều luật tương ứng?"*): Flat RAG đạt `Recall = 0.00` (`Judge = 0`), còn GraphRAG đạt `Recall = 0.67` (`Judge = 1`).
  - **Q5** (*"So sánh khung hình phạt theo luật định với mức án thực tế mà tòa tuyên cho bị cáo Phạm Trung Kiên: bị cáo bị xét xử về tội gì, Điều mấy, và mức án tòa tuyên có nằm trong khung cao nhất của điều đó không?"*): Flat RAG chỉ đạt `Recall = 0.60` (`Judge = 1`), còn GraphRAG đạt `Recall = 1.00` (`Judge = 2`).
- **Tại sao Flat RAG thất bại trên truy vấn liên-KB?**
  - Trong câu hỏi **Q3**, truy vấn chỉ chứa thực thể neo ở KB Tin tức (`"Nguyễn Văn Hùng"`), hoàn toàn **không chứa số hiệu điều luật** (`"Điều 251"`) hay tên tội danh cụ thể. Khi `HybridRetriever` tìm kiếm top-5 chunks:
    1. Các chunks của bài báo về Nguyễn Văn Hùng (`news-100260919145531789.md`) được kéo lên vì khớp tên riêng, nhưng bài báo chỉ viết bị cáo bị truy tố về tội *"Mua bán trái phép chất ma túy"* chứ **không hề trích dẫn nội dung Điều 251 hay mức án cao nhất (tử hình) trong BLHS**.
    2. Ngược lại, các chunks của Điều 251 trong KB Luật lại không chứa từ khóa `"Nguyễn Văn Hùng"`, đồng thời trong chương XX BLHS có tới 13 điều luật đều lặp đi lặp lại cụm từ *"khung hình phạt"*, *"Bộ luật Hình sự"*, khiến chunk của Điều 251 bị đẩy ra ngoài top-5 (`k=5`).
  - Hệ quả là Flat RAG rơi vào tình trạng **"mù nửa vế" (Disconnected Context)**: có bài báo hoặc có một điều luật ngẫu nhiên, nhưng không bao giờ ghép đúng cặp `(Bài báo Nguyễn Văn Hùng) + (Điều 251 BLHS)`.
- **Cơ chế giúp GraphRAG chiến thắng:**
  - Trong `GraphRAGAgent.answer()`, bước nhận diện thực thể (NER) trích được `Person = "Nguyễn Văn Hùng"`.
  - Phương thức `Neo4jGraph.context(["Nguyễn Văn Hùng"])` thực hiện duyệt đồ thị đa bước nhảy (multi-hop traversal) trong Neo4j:
    $$(\text{Person: Nguyễn Văn Hùng}) \xrightarrow{\text{:INVOLVED\_IN}} (\text{Case}) \xrightarrow{\text{:CHARGED\_WITH}} (\text{Crime: mua bán trái phép chất ma túy}) \xleftarrow{\text{:DEFINES}} (\text{Article: 251}) \xrightarrow{\text{:HAS\_CLAUSE}} (\text{Clause: 251.4 \{death: true\}})$$
  - Nhờ nút cầu nối `:Crime {name: "mua bán trái phép chất ma túy"}` được chuẩn hóa và gộp (`MERGE`) bởi `link_entity()`, đồ thị bắc cầu trực tiếp từ bị cáo trong báo sang đúng `Article 251` và các `Clause` của nó trong BLHS, bất chấp việc hai tài liệu không hề có từ khóa tên riêng chung.
  - Ở **Q4**, GraphRAG truy xuất đúng chất ma túy (`"heroin"`), tội danh và điều luật tương ứng (`Recall = 0.67` so với `0.00` của Flat RAG), chỉ thiếu duy nhất chuỗi khối lượng chính xác do cách diễn đạt khối lượng trong tóm tắt vụ án. Ở **Q5**, GraphRAG kết hợp cả `graph.context(["Phạm Trung Kiên"])` (lấy ra `Article 251` và khung `tử hình`) lẫn top-3 chunks văn bản gốc từ `HybridRetriever` (chứa mức án thực tế tòa tuyên), nhờ đó trả lời chính xác cả 5/5 facts (`Recall = 1.00`).

### 3. Nhóm `aggregation` (`Q6`)

- **Kết quả:** Với **Q6** (*"Trong toàn bộ các bản án ở kho tin tức, những bị cáo nào bị truy tố về tội danh thuộc Điều 251 Bộ luật Hình sự? Liệt kê tên bị cáo và địa phương xét xử."*), Flat RAG đạt `Recall = 0.00`, trong khi GraphRAG tăng lên `Recall = 0.33` (`Judge = 1`).
- **Phân tích nguyên nhân & giới hạn hiện tại:**
  - **Với Flat RAG (`k=5`):** Các bị cáo bị truy tố về tội Mua bán trái phép chất ma túy (thuộc Điều 251) nằm rải rác trên nhiều bài báo khác nhau trong tổng số 10 bài báo. Câu hỏi Q6 chỉ nêu `"Điều 251 Bộ luật Hình sự"`, không nêu tên bất kỳ bị cáo nào và các bài báo cũng không ghi chữ `"Điều 251"`. Do đó, top-5 chunks của Flat RAG bị lấp đầy bởi các khoản của Điều 251 trong văn bản luật và một vài đoạn báo ngẫu nhiên, hoàn toàn không thể tập hợp (aggregate) danh sách bị cáo trên toàn kho tin tức (`Recall = 0.00`).
  - **Với GraphRAG:** Nhờ truy vấn từ hạt giống `"251"` / `"Điều 251"`, `Neo4jGraph.context()` tìm ra `(:Article {number: 251})-[:DEFINES]->(:Crime {name: "mua bán trái phép chất ma túy"})` và đi ngược qua `<-[:CHARGED_WITH]-(:Case)<-[:INVOLVED_IN]-(:Person)` cùng `(:Case)-[:LOCATED_IN]->(:Location)` để gom các bị cáo thuộc Điều 251 từ nhiều bài báo khác nhau vào chung một bảng Markdown.
  - **Tại sao GraphRAG mới đạt `0.33` trên Q6?** Có 2 nguyên nhân kỹ thuật rõ ràng trong thiết kế hiện tại:
    1. Trong `Neo4jGraph.context()`, các truy vấn láng giềng đang đặt giới hạn an toàn `LIMIT 25` trên toàn bộ mẫu khớp để bảo vệ context window, trong khi một vụ án có nhiều bị cáo và nhiều khoản luật (`Article 251` có 5 khoản $\times$ nhiều `Case` $\times$ nhiều `Person`), khiến tổ hợp tích Đề-các (Cartesian product) giữa `Clause` và `Case` làm một số bị cáo bị cắt bởi `LIMIT 25`.
    2. Một số bài báo dùng thuật ngữ rút gọn ở phần phụ hoặc gán tội danh kèm hành vi phụ (ví dụ `"mua bán, tàng trữ trái phép chất ma túy"` nếu LLM không tách làm 2 phần tử trong mảng `crimes`), khiến một số nút `Case` tạo ra nút `:Crime` kép thay vì nối hết vào `:Crime {name: "mua bán trái phép chất ma túy"}`. Nếu dùng cơ chế Text-to-Cypher chuyên biệt cho câu hỏi thống kê (`MATCH (p:Person)-[:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article {number: 251}) OPTIONAL MATCH (k)-[:LOCATED_IN]->(l:Location) RETURN DISTINCT p.name, l.name`), GraphRAG sẽ đạt `Recall = 1.00`.

---

## E3. Phân tích lỗi xây dựng đồ thị (1.0 đ)

Qua kiểm tra trực tiếp đồ thị trong Neo4j Browser (`206 nodes, 388 relationships`), dưới đây là **2 lỗi thực tế** phát sinh từ quá trình LLM (`gpt-4o-mini`) trích xuất thực thể/quan hệ trong `build_graph`:

### Lỗi 1: Nút `:Substance` và `:Crime` quá chung chung hoặc biến thể tên gọi (Over-generalized / Unresolved Synonym Entities)

- **Bằng chứng cụ thể trong Neo4j:**
  Khi chạy truy vấn kiểm tra các nút `:Substance` và `:Crime` trong Neo4j:
  ```cypher
  MATCH (s:Substance) RETURN s.name ORDER BY s.name;
  MATCH (c:Crime) RETURN c.name ORDER BY c.name;
  ```
  Trong **17 nút `:Substance`**, bên cạnh các chất hóa học chuẩn khớp với BLHS (`"heroin"`, `"methamphetamine"`, `"mdma"`, `"ketamine"`, `"thuốc phiện"`), đồ thị xuất hiện các nút rất chung chung hoặc phương ngữ báo chí như:
  - `(:Substance {name: "ma túy"})`
  - `(:Substance {name: "ma túy đá"})` (thực chất là tên gọi đường phố của `"methamphetamine"`)
  - `(:Substance {name: "ma túy tổng hợp"})` (như xuất hiện ngay trong ảnh chụp `report/img/kg_my_case.png` của vụ Cái Quang Huy: cùng một vụ án vừa nối vào `"methamphetamine"`, vừa nối vào `"ma túy tổng hợp"`)
  - `(:Substance {name: "bánh heroin"})` / `(:Substance {name: "cỏ mỹ"})`
  Tương tự, trong **13 nút `:Crime`**, mặc dù chương XX BLHS có 13 điều luật, một số bài báo đề cập tội danh rút gọn như `"vận chuyển ma túy"` (thiếu cụm `"trái phép chất"`) nên không gộp (`MERGE`) được vào nút chuẩn `"vận chuyển trái phép chất ma túy"` của Điều 250.
- **Nguyên nhân:** Hàm `link_entity()` hiện tại mới thực hiện chuẩn hóa bề mặt chuỗi (chữ thường, cắt tiền tố `"tội "`, `"chất "`, gộp khoảng trắng) chứ chưa có **bảng từ điển đồng nghĩa miền pháp lý (Domain Synonym/Alias Dictionary)** hoặc bước **Entity Resolution bằng Embedding similarity**.
- **Cách khắc phục:** Bổ sung từ điển ánh xạ đồng nghĩa trong `link_entity()` cho miền ma túy (ví dụ: `"ma túy đá" -> "methamphetamine"`, `"thuốc lắc" -> "mdma"`, `"vận chuyển ma túy" -> "vận chuyển trái phép chất ma túy"`) kết hợp bước hậu kiểm bằng cosine similarity giữa embedding của thực thể mới với danh mục thực thể chuẩn trích từ BLHS (nếu $\text{sim} \ge 0.88$ thì quy về thực thể chuẩn của BLHS).

### Lỗi 2: Nhân đôi nút `:Case` cho cùng một vụ án do cấu trúc "Tin liên quan" ở cuối bài báo (Case Duplication & Hallucinated Multi-Case Extraction)

- **Bằng chứng cụ thể trong Neo4j (`report/img/kg_my_case.png`):**
  Mặc dù kho dữ liệu `data/news/` chỉ có **10 file bài báo**, đồ thị Neo4j lại sinh ra tới **15 nút `:Case`** (thể hiện rõ trong `kg_count.png`: `Case: 15`). Cụ thể, trong ảnh `report/img/kg_my_case.png`, nút `(:Person {name: "Cái Quang Huy"})` có tới **2 cạnh `:INVOLVED_IN` nối tới 2 nút `:Case` khác nhau**:
  1. `(:Case {id: "news-100260917203001265#c0", source_doc: "news-100260917203001265"})` — trích từ bài báo chính viết về vụ Cái Quang Huy vận chuyển ma túy tại Quảng Trị.
  2. `(:Case {id: "news-100260918080821054#c1", source_doc: "news-100260918080821054"})` — trích từ dòng "tin liên quan" ở cuối bài báo của Lê Minh Thành (`news-100260918080821054.md`, dòng 92), nơi VnExpress đính kèm đoạn tóm tắt ngắn 3 câu nhắc lại vụ Cái Quang Huy!
- **Nguyên nhân:** Định danh của nút `:Case` đang được gán cục bộ theo công thức `case_id = f"{doc.id}#c{idx}"`. Khi một bài báo có hộp "Tin liên quan" ở cuối nhắc tới vụ án của bài báo khác, LLM trích xuất thành phần tử thứ 2 trong mảng `cases`, và vì `doc.id` khác nhau nên Neo4j tạo thành 2 nút `:Case` độc lập cho cùng một vụ án ngoài đời thực.
- **Cách khắc phục:**
  1. **Tiền xử lý tài liệu (Data Cleaning):** Cắt bỏ các khối "Tin liên quan" / "Xem thêm" ở cuối bài báo trước khi đưa vào pipeline trích xuất KG.
  2. **Định danh `:Case` theo chữ ký thực thể (Entity-Signature Dedup):** Thay vì đặt khóa chính của `Case` theo `doc.id`, sinh khóa chuẩn hóa dựa trên tổ hợp `(tên bị cáo chính đã sort + địa phương + tội danh chính)` hoặc thực hiện bước gộp (`Case Merging`) các nút `Case` chia sẻ chung `:Person` + `:Crime` + `:Location`.

---

## E4. Khi nào KHÔNG nên dùng GraphRAG? (1.0 đ)

Dựa trực tiếp trên bảng số liệu thực nghiệm ở **Mục E1**, GraphRAG **không phải là "viên đạn bạc"** cho mọi bài toán RAG. Dưới đây là phân tích định lượng chi phí – lợi ích (Cost–Benefit Analysis) và **3 kịch bản thực tế nên chọn Flat RAG thay vì GraphRAG**:

### 1. Phân tích định lượng đánh đổi Chi phí – Độ trễ – Chất lượng từ E1

- **Chi phí và thời gian Indexing:** GraphRAG tốn **`\$0.00950`** và **143.4 giây** để nạp 11 tài liệu (1 luật + 10 báo), đắt gấp **8.48 lần** (`\$0.00950` vs `\$0.00112`) và chậm gấp **2.88 lần** (`143.4s` vs `49.8s`) so với Flat RAG. Nguyên nhân là GraphRAG phải gọi LLM sinh tới **4,993 output tokens** JSON để bóc tách thực thể/quan hệ trên từng văn bản, trong khi Flat RAG chỉ gọi API Embedding (`0` output tokens). Nếu kho tài liệu mở rộng lên 100,000 văn bản, chi phí indexing của GraphRAG sẽ đội lên hàng trăm USD và mất nhiều giờ đồng hồ.
- **Chi phí và độ trễ Querying:** Trên mỗi câu hỏi, GraphRAG tiêu thụ trung bình **4,791 input tokens** (gấp **6.90 lần** mức 694 tokens của Flat RAG), làm chi phí mỗi truy vấn đắt gấp **5.85 lần** (`\$0.00076` vs `\$0.00013`) và thời gian phản hồi chậm gấp **3.67 lần** (**5.73 giây** vs **1.56 giây**), do phải tốn **2 lần gọi LLM tuần tự** (Lần 1: NER trích thực thể hạt giống từ câu hỏi; Lần 2: Tổng hợp câu trả lời từ bảng đồ thị + văn bản).
- **Lợi ích ròng theo loại câu hỏi:**
  - Trên các câu hỏi `cross-kb` / `multi-hop` (Q3, Q4, Q5), việc trả thêm **5.85× chi phí** là hoàn toàn xứng đáng vì Recall tăng vọt từ `0.00–0.60` lên `0.67–1.00`.
  - Ngược lại, trên các câu hỏi `single-hop` (Q1, Q2), **lợi ích ròng về độ chính xác bằng 0** (`Recall` cùng đạt `1.00`, `Judge` cùng đạt `2/2`), nghĩa là dùng GraphRAG ở đây chỉ làm lãng phí **85% chi phí** và khiến người dùng phải chờ lâu hơn gấp **3.7 lần**.

### 2. Ba kịch bản thực tế nên chọn Flat RAG thay vì GraphRAG

1. **Kịch bản 1 — Hệ thống hỏi đáp tra cứu trực tiếp 1 bước nhảy (Single-hop FAQ / Policy Lookup) với yêu cầu độ trễ thời gian thực (Real-time SLA < 2 giây):**
   - *Ví dụ:* Chatbot CSKH tra cứu quy định đổi trả hàng, tra cứu từng điều khoản hợp đồng hoặc tra cứu thông tin một bản tin cụ thể (tương tự **Q1** và **Q2**).
   - *Lý do chọn Flat RAG:* Mọi thông tin cần thiết đã nằm trọn trong 1–2 đoạn văn bản liền kề. Flat RAG đạt `Recall = 1.00` với độ trễ chỉ **1.56s** (đáp ứng SLA < 2s của voicebot/livechat) và chi phí chỉ `\$0.00013`/câu, trong khi GraphRAG mất **5.73s** và đắt gấp gần 6 lần mà không tăng thêm độ chính xác.
2. **Kịch bản 2 — Kho dữ liệu biến động liên tục theo thời gian thực và quy mô lớn (High-churn, High-volume Streaming Corpus):**
   - *Ví dụ:* Hệ thống tổng hợp tin tức thời sự hàng phút (hàng chục nghìn bài báo/ngày), log hệ thống hoặc ticket hỗ trợ khách hàng liên tục cập nhật.
   - *Lý do chọn Flat RAG:* Với chi phí indexing đắt gấp **8.48 lần** và thời gian nạp lâu gấp **2.88 lần** (`143.4s` cho chỉ 11 văn bản), việc chạy LLM trích xuất đồ thị cho hàng chục nghìn văn bản mới mỗi ngày sẽ gây nghẽn cổ chai (ingestion bottleneck) và tốn kém chi phí API khổng lồ, chưa kể rủi ro rác đồ thị do lỗi trích xuất (như đã thấy ở mục E3). Flat RAG chỉ cần băm chunk và tính vector embedding cực nhanh, dễ dàng thêm/xóa từng văn bản độc lập.
3. **Kịch bản 3 — Miền dữ liệu phi cấu trúc không có lược đồ thực thể – quan hệ rõ ràng (Schema-poor / Narrative & Conceptual Texts):**
   - *Ví dụ:* Kho tài liệu hướng dẫn kỹ năng mềm, bài giảng lý thuyết, văn học, bình luận triết học hoặc tài liệu thiết kế UX.
   - *Lý do chọn Flat RAG:* GraphRAG chỉ phát huy sức mạnh khi miền dữ liệu có **Ontology chặt chẽ** và các **nút cầu nối định danh rõ ràng** (như `Crime`, `Substance`, `Article` trong luật hình sự). Với văn bản thiên về lập luận trừu tượng hoặc mô tả quy trình tự nhiên, việc ép dữ liệu vào các bộ ba `(Subject)-[:PREDICATE]->(Object)` vừa làm mất ngữ cảnh tinh tế của câu văn, vừa khiến bước NER/Entity Linking thất bại, dẫn tới kết quả kém hơn cả tìm kiếm vector + BM25 truyền thống.
