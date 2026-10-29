# KẾ HOẠCH TRIỂN KHAI DATA INDEXING — DỰ ÁN MULTIMODAL HYBRID RETRIEVAL

> **Phụ trách:** Quân — Giai đoạn 0 (Data Indexing / Offline)  
> **Mục tiêu tài liệu:** Dùng như checklist để hoàn thành từng bước, không phải tài liệu code.  
> **Trạng thái:** **BƯỚC 00 đã hoàn thành ở mức chốt thiết kế (29/10/2026)**; các bước 01–12 chưa được xác minh triển khai. Chỉ đổi `[ ]` thành `[x]` khi công việc tương ứng có minh chứng.  
> **Phạm vi:** Chi tiết phần Data Indexing của Quân; cuối tài liệu có các mốc phối hợp với toàn nhóm.

## 1. Mục tiêu cuối cùng

Xây dựng **corpus tri thức tiếng Anh về Plants and Animals từ Wikipedia** để hỗ trợ Retrieval Service tìm kiếm theo hai cách:

1. **Sparse retrieval (BM25 / Lucene):** tra cứu theo từ khóa.
2. **Dense retrieval (BGE-M3 / Qdrant):** tra cứu theo ý nghĩa ngữ nghĩa.

Nội dung đoạn văn và metadata được lưu tại **PostgreSQL**. Cùng một `chunk_id` phải ánh xạ đến **cùng một đoạn văn** ở PostgreSQL, Lucene và Qdrant. Đồng thời, chuẩn bị **tập câu hỏi OK-VQA thuộc Plants and Animals** cho bước đánh giá, nhưng **không đưa câu hỏi/đáp án OK-VQA vào corpus tìm kiếm**.

### Sơ đồ phụ thuộc

```text
Wikipedia (Plants and Animals)
          |
          v
    Collect / Clean
          |
          v
  Chunk (~200 từ, overlap 40)
          |
          v
 PostgreSQL: documents + chunks
          |
          +--------------------------+
          |                          |
          v                          v
  BM25 / Lucene               BGE-M3 embeddings
  Sparse Index                       |
                                     v
                              Qdrant Collection
          |                          |
          +------------+-------------+
                       |
                       v
          Validate + Bàn giao cho
             Retrieval Service

OK-VQA (Plants and Animals) ---> Tập đánh giá RIÊNG (không lập chỉ mục)
```

### Tiêu chí hoàn thành toàn bộ phần việc

- [ ] Có dữ liệu Wikipedia đã xử lý, có nguồn và metadata.
- [ ] Có tập OK-VQA đã lọc riêng phục vụ đánh giá.
- [ ] Có bảng `documents`, `chunks` trong PostgreSQL chứa dữ liệu đúng cấu trúc đã thống nhất.
- [ ] Có Lucene Index truy vấn BM25 được.
- [ ] Có Qdrant Collection chứa embeddings và tìm kiếm vector được.
- [ ] `chunk_id` nhất quán trên cả ba hệ thống.
- [ ] Có kiểm thử tự động/thủ công với đầu vào, đầu ra rõ ràng.
- [ ] Có thể chạy lại quy trình từ dữ liệu nguồn bằng các script và cấu hình đã lưu.
- [ ] Thành viên Retrieval Service nhận được dữ liệu mẫu và hướng dẫn kết nối.

## 2. Bảng theo dõi tiến độ

| Bước | Công việc | Mức ưu tiên | Trạng thái |
|---|---|---|---|
| 00 | Chốt hợp đồng dữ liệu và các lựa chọn kỹ thuật | Bắt buộc trước khi code phụ thuộc | ✅ Đã chốt thiết kế |
| 01 | Chuẩn bị thư mục, môi trường và Git | Cao | ⬜ |
| 02 | Tải và lọc OK-VQA phục vụ đánh giá | Cao, có thể làm song song | ⬜ |
| 03 | Thu thập Wikipedia, bắt đầu từ mẫu nhỏ | Cao | ⬜ |
| 04 | Tiền xử lý và loại trùng | Cao | ⬜ |
| 05 | Chunking và chuẩn hóa `chunk_id` | Cao | ⬜ |
| 06 | Lưu tài liệu/chunk vào PostgreSQL | Cao | ⬜ |
| 07 | Tạo và kiểm tra BM25 / Lucene Index | Cao | ⬜ |
| 08 | Tạo và kiểm tra embeddings / Qdrant Index | Cao | ⬜ |
| 09 | Kiểm thử đồng bộ, dữ liệu và truy xuất | Cao | ⬜ |
| 10 | Tích hợp thử với Retrieval Service | Cao | ⬜ |
| 11 | Mở rộng corpus và kiểm tra chất lượng | Sau khi mẫu nhỏ chạy ổn | ⬜ |
| 12 | Hoàn thiện tài liệu, bàn giao | Cuối giai đoạn | ⬜ |

> **Gợi ý cách dùng:** Bắt đầu triển khai ở **BƯỚC 01**, rồi kiểm tra điều kiện hoàn thành trước khi sang bước tiếp theo. Các quyết định ở BƯỚC 00 có hiệu lực cho tất cả module liên quan; việc triển khai, pin phiên bản và kiểm chứng kỹ thuật vẫn phải thực hiện ở các bước sau. Nếu phát hiện vướng mắc mới, ghi vào mục 6.

---

## 3. Checklist triển khai chi tiết

### BƯỚC 00 — Chốt hợp đồng dữ liệu và cấu hình kỹ thuật (ĐÃ HOÀN THÀNH THIẾT KẾ)

**Mục tiêu:** Có một hợp đồng thống nhất để Data Indexing, HyDE Server và Retrieval Service làm theo, không phụ thuộc việc các module được lập trình ở các máy khác nhau.

**Người chốt:** Quân (người quyết định cấu hình dự án). **Ngày chốt:** 29/10/2026. Đây là **quyết định kỹ thuật**, không phải xác nhận rằng hạ tầng/model đã được cài hoặc chạy thử.

**Checklist BƯỚC 00**

- [x] Chốt phạm vi **tiếng Anh**, miền **Plants and Animals**.
- [x] Chốt **Wikipedia** là corpus truy xuất chính; NaturalQuestionsV2 chỉ dùng bổ sung nếu cần; **OK-VQA dành riêng cho đánh giá**, không chép câu hỏi/đáp án vào corpus.
- [x] Chốt **BGE-M3**, embedding dense 1.024 chiều, quy tắc chuẩn hóa và thư viện dùng chung với HyDE.
- [x] Chốt quy tắc `document_id`, `chunk_index`, `chunk_id`, ánh xạ sang Lucene và Qdrant.
- [x] Chốt schema PostgreSQL tối thiểu cho `documents` và `chunks`.
- [x] Chốt tên Qdrant Collection, vector size, distance và payload.
- [x] Chốt cơ chế Retrieval Service truy cập PostgreSQL, Qdrant và Lucene Index.
- [x] Ghi các quyết định thành hợp đồng kỹ thuật tại ngay BƯỚC 00 của file này để các module dùng chung; việc tạo/sao chép tài liệu tại `docs/architecture.md` và `docs/api.md` được theo dõi trong BƯỚC 01 và khi tích hợp.

#### 00.1. Nguồn dữ liệu và phân đoạn

| Hạng mục | Quyết định chính thức |
|---|---|
| Corpus tìm kiếm | Các bài Wikipedia **tiếng Anh** về **Plants and Animals** |
| Nguồn bổ sung | NaturalQuestionsV2: tùy chọn, không cần tải toàn bộ |
| Tập đánh giá | OK-VQA, lọc các câu hỏi thuộc Plants and Animals; lưu **tách biệt** với corpus |
| Chunking | Mục tiêu **200 từ/chunk**, overlap **40 từ**; quy tắc tách từ/đoạn cuối được kiểm thử ở BƯỚC 05 |
| Quy mô | Pipeline mẫu vài chục bài → khoảng **5.000 chunks** → cân nhắc **20.000–50.000 chunks** |

#### 00.2. Text Encoder dùng chung cho Data Indexing và HyDE

| Thông số | Quyết định chính thức |
|---|---|
| Model | `BAAI/bge-m3` |
| Thư viện chính | `FlagEmbedding` |
| Dạng embedding | Chỉ dùng **dense embedding** của BGE-M3 cho Qdrant; không nhầm sang sparse / ColBERT multi-vector |
| Số chiều | **1024** |
| Chuẩn hóa | **Có**, dùng vector đã chuẩn hóa (L2) |
| Kiểu dữ liệu lưu vector | `float32` |
| Tiền xử lý đầu vào | Văn bản tiếng Anh đã làm sạch; **không tự thêm prefix/prompt khác nhau** giữa document và `P_hypo` |
| Pooling | Dùng **đầu ra dense chuẩn của BGE-M3 qua FlagEmbedding**, không tự thay bằng một cách pooling khác |
| Checkpoint / dependencies | **Phải pin một revision/commit hash thực tế và phiên bản thư viện khi cài ở BƯỚC 01, trước khi tạo Dense Index**; chưa có hash đã tải nên **không ghi giả** |
| Kiểm thử tương thích | Ở BƯỚC 08/10: dùng cùng cấu hình encode một chuỗi mẫu trên hai module; kiểm tra số chiều và độ lệch vector |

**Quy tắc không được thay đổi khi tích hợp:** HyDE Server phải tạo `V_query` bằng **cùng checkpoint và quy trình embedding** được dùng để mã hóa Wikipedia. Cùng 1.024 chiều **chưa đủ** để bảo đảm tương thích. Nếu cần đổi model, phải xây lại Dense Index/Collection có phiên bản phù hợp.

#### 00.3. Hợp đồng ID và PostgreSQL

**Schema tối thiểu:**

```text
documents(document_id, title, source, url, domain)
chunks(chunk_id, document_id, text, chunk_index)
```

| Trường / hệ thống | Quy tắc chính thức |
|---|---|
| `document_id` | Chuỗi `wiki_en_<page_id>`, dùng **page ID** của Wikipedia tiếng Anh. Ví dụ: `wiki_en_12345` |
| `chunk_index` | Số nguyên không âm, bắt đầu từ `0` trong mỗi tài liệu |
| `chunk_id` | `<document_id>_v1_chunk_<chunk_index:04d>`. Ví dụ: `wiki_en_12345_v1_chunk_0000`; tối thiểu 4 chữ số, không giới hạn tối đa 9.999 chunks |
| Phiên bản corpus | Hậu tố `_v1_` nằm trong ID; khi thay đổi nội dung/chunking làm cùng ID có thể trỏ đến đoạn khác, tăng phiên bản và rebuild đồng bộ |
| PostgreSQL | `documents.document_id` là PK; `chunks.chunk_id` là PK; `chunks.document_id` là FK; thêm UNIQUE (`document_id`, `chunk_index`) trong từng bộ corpus/phiên bản tương ứng |
| Lucene | Dùng **chuỗi `chunk_id`** làm document ID của chỉ mục; kết quả BM25 phải ánh xạ ngược về PostgreSQL |
| Qdrant | **Point ID UUIDv5** từ `chunk_id`, đồng thời lưu `chunk_id` trong payload; không dùng chuỗi `chunk_id` tùy ý làm point ID |

**Quy ước UUIDv5 để tái tạo nhất quán:** Dùng namespace chuẩn `UUID_NAMESPACE_URL` (tương đương `uuid.NAMESPACE_URL` trong Python) và tên chính xác `okvqa-data-indexing:wikipedia_chunks_v1:<chunk_id>`; thay `<chunk_id>` bằng giá trị chuỗi thực. Như vậy cùng một `chunk_id` luôn tạo cùng một Qdrant point ID.

**Lưu ý:** `document_id` nêu trên dành cho bài Wikipedia; nếu sau này nhập nguồn không phải Wikipedia, phải dùng namespace khác để tránh xung đột. Định dạng trường mở rộng và triển khai migration nằm ở BƯỚC 06.

#### 00.4. Qdrant Collection

| Thông số | Quyết định chính thức |
|---|---|
| Collection | `wikipedia_chunks_v1` |
| Vector size | `1024` |
| Distance | `Cosine` |
| Point ID | UUIDv5 theo quy tắc ở mục 00.3 |
| Payload bắt buộc | `chunk_id`, `document_id`, `title`, `domain`, `source` |
| Nội dung text chuẩn và URL | Đọc từ **PostgreSQL**, thông qua `chunk_id`; payload không thay PostgreSQL làm nguồn chuẩn |

#### 00.5. Hạ tầng và cách các module truy cập dữ liệu

| Thành phần | Cách dùng chính thức | Biến cấu hình |
|---|---|---|
| PostgreSQL | Chạy local qua **Docker Compose** ở giai đoạn phát triển; Retrieval Service kết nối bằng URL | `DATABASE_URL` |
| Qdrant | Chạy local qua **Docker Compose** ở giai đoạn phát triển; Retrieval Service gọi HTTP endpoint | `QDRANT_URL`, `QDRANT_COLLECTION` (và `QDRANT_API_KEY` nếu môi trường yêu cầu) |
| Lucene/Pyserini | Index là **thư mục trên ổ đĩa**. Retrieval Service đọc thư mục được mount/copy hoặc artifact versioned, **không mặc định có API HTTP** | `LUCENE_INDEX_PATH` |
| Embedding | Cả Data Indexing và HyDE phải đọc cùng model ID, revision và quy tắc encode | `EMBEDDING_MODEL_ID`, `EMBEDDING_MODEL_REVISION` |

- Các dịch vụ có thể chạy trên máy khác bằng cách cấu hình endpoint có thể truy cập và được bảo vệ; Lucene Index phải được **bàn giao/copy kèm phiên bản** hoặc dùng shared volume.
- Không commit `.env`, token, password, dữ liệu lớn hoặc thư mục index lên Git. Dùng `.env.example` với giá trị giả.
- Các API HyDE/Retrieval/VQA theo `project-plan.md`; **Data Indexing là chương trình offline, không tự tạo thêm API** nếu chưa có nhu cầu.

**Điều kiện hoàn thành BƯỚC 00: ĐÃ ĐẠT (thiết kế).** Các lựa chọn, schema, quy tắc ID và phương thức kết nối đã được chốt tại đây. **Chưa kiểm chứng triển khai**: pin revision/version thực tế, tạo container, test embedding, tạo collection/index, đồng bộ dữ liệu và viết tài liệu tích hợp vẫn là công việc của BƯỚC 01/06–10. Việc các thành viên cần làm là **tuân theo hợp đồng này**, không phải bỏ phiếu xác nhận lại.

### BƯỚC 01 — Chuẩn bị cấu trúc dự án và môi trường

**Mục tiêu:** Tạo nền tảng làm việc độc lập, có thể chạy và kiểm thử từng script.

**Công việc**

- [ ] Làm việc trong `data-indexing/` của monorepo; kiểm tra nhánh Git phụ trách theo quy ước nhóm.
- [ ] Chuẩn bị `src/`, `tests/`, `README.md`, `requirements.txt`, `.env.example`.
- [ ] Tạo môi trường Python riêng cho module và ghi cách kích hoạt trong README.
- [ ] Chuẩn bị **Docker Compose local** cho PostgreSQL và Qdrant theo hợp đồng ở BƯỚC 00; thử kết nối bằng các biến môi trường mẫu (không đẩy secret lên Git).
- [ ] Quy định nơi lưu `data/raw`, `data/processed`, `data/indexes`.
- [ ] Cấu hình `.gitignore` để không đẩy `.env`, credential, file dữ liệu lớn, thư mục môi trường ảo hoặc dữ liệu chỉ mục không cần quản lý phiên bản.
- [ ] Pin phiên bản `FlagEmbedding`/dependencies và **revision/commit hash thực tế** của `BAAI/bge-m3`; ghi vào cấu hình/README để Data Indexing và HyDE cùng sử dụng (không tự ghi revision khi chưa tải model).
- [ ] Sao chép hợp đồng ở BƯỚC 00 vào `docs/architecture.md`; ghi giao diện liên quan trong `docs/api.md` nếu cần.
- [ ] Tạo một commit khởi tạo module và kiểm tra người khác có thể clone, làm theo README.

**Điều kiện hoàn thành:** Cấu trúc thư mục rõ ràng; môi trường chạy được; cấu hình mẫu không chứa bí mật thật; Git sạch và có hướng dẫn tái lập.

### BƯỚC 02 — Chuẩn bị tập OK-VQA để đánh giá

**File dự kiến:** `src/download_okvqa.py`

**Mục tiêu:** Tách một tập câu hỏi/ảnh/đáp án chuẩn cho đánh giá, **không dùng làm nguồn corpus**.

**Công việc**

- [ ] Xác nhận đường dẫn và định dạng dữ liệu OK-VQA được nhóm sử dụng.
- [ ] Tải/đọc các tập cần thiết theo kế hoạch đánh giá (tài liệu nêu train và validation trước khi lọc).
- [ ] Kiểm tra metadata thực tế để xác định chính xác cách lọc nhóm **Plants and Animals**; không tự giả định tên cột/nhãn.
- [ ] Lọc các câu hỏi thuộc nhóm đã chọn, bảo toàn liên kết câu hỏi–ảnh–đáp án.
- [ ] Lưu dữ liệu đã lọc vào khu vực đánh giá riêng với các trường cần thiết đã thống nhất.
- [ ] Thống kê số mẫu ban đầu, số mẫu sau lọc, số mẫu thiếu ảnh/đáp án/metadata.
- [ ] Kiểm tra một số mẫu ngẫu nhiên bằng cách đối chiếu câu hỏi và ảnh.
- [ ] Bảo đảm **không trộn câu hỏi hoặc đáp án OK-VQA vào tập đoạn Wikipedia để lập chỉ mục**.

**Điều kiện hoàn thành:** Có tập OK-VQA đã lọc, thống kê, quy trình tái lập và kiểm tra được quan hệ ảnh–câu hỏi–đáp án.

### BƯỚC 03 — Thu thập Wikipedia: làm mẫu nhỏ trước

**File dự kiến:** `src/collect_wikipedia.py`

**Mục tiêu:** Có một corpus Wikipedia tiếng Anh phù hợp với Plants and Animals, truy xuất được nguồn gốc.

**Công việc**

- [ ] Chọn phương thức thu thập Wikipedia thích hợp và ghi rõ cách chọn bài.
- [ ] Trước tiên thu thập **mẫu nhỏ** (đề xuất: vài chục bài) để kiểm tra toàn bộ quy trình trước khi tăng quy mô.
- [ ] Bảo đảm bài viết liên quan đến thực vật/động vật; tránh lấy nhầm miền chủ đề.
- [ ] Lưu các thông tin ít nhất: tiêu đề, URL/nguồn, miền tri thức và nội dung gốc.
- [ ] Kiểm tra lỗi thu thập: nội dung rỗng, sai ngôn ngữ, lỗi kết nối, bài lặp, bản ghi thiếu URL.
- [ ] Lưu bản thô trong `data/raw/` và ghi số lượng, thời điểm/phiên bản thu thập.
- [ ] Sau khi pipeline mẫu chạy ổn mới tăng khối lượng thu thập.

**Điều kiện hoàn thành:** Tập mẫu có nội dung hợp lệ, nguồn rõ ràng và có thể thu thập lại.

### BƯỚC 04 — Làm sạch và loại bỏ trùng lặp

**File dự kiến:** `src/preprocess.py`

**Mục tiêu:** Tạo tài liệu văn bản sạch trước khi chia chunk.

**Công việc**

- [ ] Xác định nội dung nào cần giữ và loại bỏ (ví dụ: phần điều hướng, markup thừa, đoạn trống; quyết định theo định dạng nguồn thực tế).
- [ ] Chuẩn hóa khoảng trắng, ngắt dòng, mã hóa Unicode mà không làm mất nội dung có nghĩa.
- [ ] Loại bản ghi rỗng, bất thường và trùng lặp; xác định rõ tiêu chí phát hiện trùng.
- [ ] Giữ nguyên thông tin liên kết từ văn bản sau xử lý về `document_id`, tiêu đề, URL nguồn.
- [ ] Kiểm thử trên một vài tài liệu ngắn/dài, nhiều đoạn, có ký tự đặc biệt.
- [ ] Tạo báo cáo trước/sau tiền xử lý: số bài, số bản ghi bị loại, các lý do loại.

**Điều kiện hoàn thành:** Corpus đã sạch và không bị mất metadata hoặc biến dạng nội dung quan trọng; chạy lại cho kết quả ổn định.

### BƯỚC 05 — Chunking và đặt ID nhất quán

**File dự kiến:** `src/chunking.py`

**Mục tiêu:** Chuyển từng bài Wikipedia thành nhiều đoạn văn để lập chỉ mục.

**Công việc**

- [ ] Áp dụng cấu hình khởi đầu của dự án: **khoảng 200 từ/chunk, overlap 40 từ**.
- [ ] Quy định rõ cách đếm từ, xử lý đoạn cuối ngắn, và không tạo chunk rỗng.
- [ ] Gán `document_id = wiki_en_<page_id>`, `chunk_index` từ 0, `chunk_id = <document_id>_v1_chunk_<chunk_index:04d>` theo BƯỚC 00; kiểm tra không trùng lặp.
- [ ] Liên kết mỗi chunk với văn bản, tiêu đề, URL và domain của tài liệu nguồn.
- [ ] Lưu tập chunk đã xử lý ở `data/processed/` với định dạng đã thống nhất.
- [ ] Kiểm tra thủ công các chunk liên tiếp để chắc rằng phần overlap đúng và không bỏ sót nội dung.
- [ ] Kiểm tra tự động tính duy nhất `chunk_id` và số lượng chunk theo tài liệu.

**Điều kiện hoàn thành:** Mỗi chunk có ID duy nhất và truy ngược được tài liệu gốc; số lượng chunk hợp lý; overlap hoạt động đúng.

### BƯỚC 06 — Lưu vào PostgreSQL

**File dự kiến:** `src/database.py`

**Mục tiêu:** PostgreSQL là nơi chứa nội dung tài liệu/chunk và metadata để Retrieval Service lấy văn bản chính xác.

**Công việc**

- [ ] Tạo schema `documents` và `chunks` theo hợp đồng đã thống nhất.
- [ ] Khai báo khóa chính cho `document_id`, `chunk_id`; liên kết `chunks.document_id` với tài liệu gốc.
- [ ] Viết tiến trình nhập dữ liệu từ corpus/chunks đã xử lý.
- [ ] Đảm bảo việc chạy lại không vô tình tạo bản ghi lặp hoặc ghi sai mapping.
- [ ] Kiểm tra số lượng documents và chunks trước/sau khi nhập.
- [ ] Truy vấn thử bằng một `chunk_id` để xem lấy được đúng `text`, `title` và `url`.
- [ ] Ghi cách cấu hình kết nối database và cách chuẩn bị schema trong README.

**Điều kiện hoàn thành:** PostgreSQL có dữ liệu chính xác, liên kết khóa hợp lệ và tra cứu một chunk bằng ID hoạt động.

### BƯỚC 07 — Xây dựng Sparse Index (BM25 / Lucene)

**File dự kiến:** `src/build_sparse_index.py`

**Mục tiêu:** Tìm kiếm đoạn văn theo từ khóa bằng BM25.

**Công việc**

- [ ] Sử dụng **Pyserini/Lucene (BM25)** theo hợp đồng; chốt analyzer/cấu hình lập chỉ mục thực tế và ghi trong README.
- [ ] Chuẩn bị dữ liệu đầu vào cho công cụ lập chỉ mục: ít nhất gồm ID chunk và nội dung văn bản.
- [ ] Xây dựng Lucene Index từ cùng tập chunk đã nhập PostgreSQL.
- [ ] Lưu rõ đường dẫn và tham số lập chỉ mục.
- [ ] Chạy vài truy vấn tiếng Anh có từ khóa cụ thể và xem Top-K trả về.
- [ ] Kiểm tra mỗi kết quả có `chunk_id` hợp lệ và tra được đúng chunk trong PostgreSQL.
- [ ] Ghi cách xây lại index và cách mở index cho Retrieval Service.

**Điều kiện hoàn thành:** BM25 trả được danh sách chunk theo thứ hạng; ID không sai; index có thể được tái tạo từ dữ liệu.

### BƯỚC 08 — Xây dựng Dense Index (BGE-M3 / Qdrant)

**File dự kiến:** `src/build_dense_index.py`

**Mục tiêu:** Mã hóa cùng tập chunk thành vector và lưu vào Qdrant để tìm kiếm ngữ nghĩa.

**Công việc**

- [ ] Đối chiếu model ID, **revision đã pin ở BƯỚC 01**, phiên bản `FlagEmbedding`, cách chuẩn hóa và đầu ra dense BGE-M3 với hợp đồng ở BƯỚC 00; không thay đổi cấu hình tùy ý.
- [ ] Tải/khởi tạo Text Encoder đã thống nhất; kiểm tra số chiều vector (dự kiến BGE-M3: **1.024**).
- [ ] Sinh embedding thử cho một tập chunk nhỏ, kiểm tra có đủ số vector và không có vector lỗi.
- [ ] Tạo Qdrant Collection `wikipedia_chunks_v1` với vector size **1024**, distance **Cosine**, payload và quy tắc UUIDv5 ở BƯỚC 00.
- [ ] Ghi vector kèm khóa/mapping để truy ngược đúng `chunk_id` và metadata cần thiết.
- [ ] Nhập embeddings theo lô phù hợp với tài nguyên máy; có cơ chế xử lý nếu lô bị lỗi.
- [ ] Kiểm tra số vector Qdrant và số chunk PostgreSQL có khớp với phạm vi lập chỉ mục hay không.
- [ ] Tạo một query embedding mẫu bằng **cùng encoder**, thử tìm Top-K và đối chiếu các ID nhận được.
- [ ] Ghi lại model, cấu hình embedding, collection và cách xây lại dense index.

**Điều kiện hoàn thành:** Qdrant tìm kiếm vector được, số chiều đúng, không thiếu/nhầm chunk; model tạo query tương thích với model tạo document embedding.

### BƯỚC 09 — Kiểm thử xuyên suốt dữ liệu và hai index

**File dự kiến:** `src/validate_indexes.py` và các test trong `tests/`.

**Mục tiêu:** Bảo đảm module Data Indexing thực sự sẵn sàng phục vụ Retrieval Service.

**Công việc**

- [ ] Đối chiếu số lượng và tập `chunk_id` trong PostgreSQL, Lucene, Qdrant.
- [ ] Với một số `chunk_id` lấy từ từng index, truy ngược PostgreSQL và so sánh nội dung.
- [ ] Kiểm tra ID trùng, chunk rỗng, bản ghi mồ côi, bản ghi thiếu metadata.
- [ ] Kiểm tra vector đúng số chiều và không chứa giá trị không hợp lệ.
- [ ] Thử BM25 với câu truy vấn chứa từ khóa đặc trưng và Dense Search với truy vấn ngữ nghĩa; đọc kết quả thực tế để đánh giá sơ bộ.
- [ ] Kiểm tra kết quả khi dữ liệu/truy vấn rỗng hoặc hệ thống lưu trữ chưa sẵn sàng.
- [ ] Kiểm tra chạy lại các script có làm hỏng index hoặc nhân đôi dữ liệu không.
- [ ] Lưu báo cáo kiểm thử: tổng số chunks, số lỗi, một số truy vấn mẫu và kết quả.

**Điều kiện hoàn thành:** Những lỗi sai lệch mapping và lỗi dữ liệu quan trọng đã được xử lý; có kết quả kiểm thử chứng minh.

### BƯỚC 10 — Kiểm tra tích hợp với Retrieval Service

**Mục tiêu:** Bên Retrieval có thể sử dụng trực tiếp cả ba nguồn dữ liệu mà không phải chuyển đổi lại.

**Công việc**

- [ ] Chuyển cho thành viên Retrieval Service cấu trúc bảng PostgreSQL và thông số kết nối được cung cấp qua kênh phù hợp (không commit mật khẩu).
- [ ] Chuyển cách truy cập Lucene Index, Qdrant Collection và thông tin `chunk_id`.
- [ ] Cung cấp vài `chunk_id`, từ khóa truy vấn và kết quả tham chiếu để test.
- [ ] Phối hợp thử đường đi: `P_hypo` → BM25 và `V_query` → Dense → lấy đúng văn bản theo ID.
- [ ] Xác minh vector từ HyDE Server dùng cùng **model revision, cách chuẩn hóa, cấu hình encode**, đúng 1024 chiều; thử encode cùng chuỗi và kiểm tra tính tương thích.
- [ ] Xử lý các lỗi giao diện phát hiện trong quá trình tích hợp, cập nhật tài liệu chung.

**Điều kiện hoàn thành:** Retrieval Service truy xuất được dữ liệu thật từ Lucene và Qdrant, sau đó lấy được đúng passage trong PostgreSQL; hai bên xác nhận giao diện tương thích.

### BƯỚC 11 — Mở rộng corpus và tối ưu sau khi đã chạy mẫu thành công

**Mục tiêu:** Chuyển từ bản chạy thử nhỏ sang corpus đủ lớn để phục vụ thí nghiệm.

**Công việc**

- [ ] Sau mẫu vài chục bài, tăng dần quy mô hướng tới **khoảng 5.000 chunks** để kiểm thử theo kế hoạch.
- [ ] Ghi nhận tỷ lệ tài liệu/chunk bị loại và vấn đề chất lượng nội dung.
- [ ] Đánh giá chất lượng truy xuất trên các câu hỏi hoặc truy vấn mẫu, tránh chỉ đánh giá bằng số lượng chunk.
- [ ] Theo dõi thời gian lập chỉ mục, dung lượng lưu trữ, giới hạn RAM/CPU và độ ổn định của quá trình.
- [ ] Nếu phù hợp, mở rộng tới **20.000–50.000 chunks** như định hướng tài liệu; không mở rộng bằng mọi giá khi corpus còn nhiễu.
- [ ] Xây lại hoặc cập nhật đồng bộ PostgreSQL, Lucene và Qdrant theo quy trình đã kiểm thử.
- [ ] Ghi lại phiên bản corpus và phiên bản indexes để nhóm tái lập thí nghiệm.

**Điều kiện hoàn thành:** Corpus đạt quy mô nhóm quyết định, được kiểm tra chất lượng và các index đồng bộ; có thông tin để tái lập.

### BƯỚC 12 — Viết tài liệu, chuẩn bị bàn giao

**Mục tiêu:** Người khác có thể tự chạy lại module mà không cần Quân giải thích bằng lời.

**Công việc**

- [ ] Hoàn thiện `data-indexing/README.md`: mục tiêu, kiến trúc, yêu cầu hệ thống, môi trường, dependencies.
- [ ] Ghi rõ thứ tự chạy: thu thập → tiền xử lý → chunking → PostgreSQL → Lucene/Qdrant → validation.
- [ ] Hoàn thiện `requirements.txt`, `.env.example` và hướng dẫn cấu hình không lộ secret.
- [ ] Ghi đường dẫn dữ liệu, cách tái tạo dữ liệu, cách xử lý lỗi thường gặp.
- [ ] Cung cấp ví dụ input/output của các script, các test và kết quả validation.
- [ ] Chuẩn bị mẫu dữ liệu cho thành viên Retrieval Service và tài liệu schema/mapping ID.
- [ ] Commit thay đổi trên nhánh phụ trách, tạo Pull Request vào `dev` theo quy trình nhóm.

**Điều kiện hoàn thành:** Có bộ bàn giao rõ ràng; thành viên khác làm theo README có thể khởi tạo và kiểm thử được Data Indexing.

---

## 4. Các mốc kiểm tra (milestones)

| Mốc | Khi nào đạt? | Minh chứng cần lưu |
|---|---|---|
| **M0 — Contract Ready** | Xong bước 00–01 | Quyết định schema/model/ID, môi trường, repo |
| **M1 — Data Ready** | Xong bước 02–05 | OK-VQA tách riêng, Wikipedia mẫu sạch, chunks có ID |
| **M2 — Storage Ready** | Xong bước 06 | Bảng documents/chunks có dữ liệu, truy vấn ID thành công |
| **M3 — Dual Index Ready** | Xong bước 07–08 | BM25 và Dense đều trả Top-K với ID hợp lệ |
| **M4 — Retrieval-Ready MVP** | Xong bước 09–10 | Báo cáo validation, Retrieval Service đọc được dữ liệu |
| **M5 — Handover Complete** | Xong bước 11–12 | Corpus theo quy mô thống nhất, scripts, README, PR |

**Thứ tự ưu tiên thực tế:** Hoàn thiện **một pipeline nhỏ nhưng chạy xuyên suốt** trước, rồi mới mở rộng dữ liệu. Không cần đợi tải đủ hàng chục nghìn chunks mới kiểm thử Lucene, Qdrant hoặc PostgreSQL.

## 5. Cấu trúc file dự kiến (theo kế hoạch nhóm)

```text
data-indexing/
├── src/
│   ├── download_okvqa.py       # Tải/lọc OK-VQA để đánh giá
│   ├── collect_wikipedia.py    # Thu thập Wikipedia
│   ├── preprocess.py           # Làm sạch, loại trùng
│   ├── chunking.py             # Chia đoạn, tạo ID
│   ├── database.py             # PostgreSQL
│   ├── build_sparse_index.py   # BM25 / Lucene
│   ├── build_dense_index.py    # Embedding / Qdrant
│   └── validate_indexes.py    # Kiểm thử mapping & index
├── tests/
├── README.md
├── requirements.txt
└── .env.example

# Vị trí data theo monorepo chung:
data/
├── raw/
├── processed/
└── indexes/
```

Đây là **cấu trúc dự kiến**, không phải xác nhận rằng các file đã tồn tại. Theo tài liệu, có thể gộp hoặc bổ sung file nếu có lý do phù hợp.

## 6. Sổ quyết định kỹ thuật và nhật ký vướng mắc

> **Nguyên tắc:** Quân đã **chốt các lựa chọn kiến trúc ở BƯỚC 00**. Những mục đánh dấu “cần kiểm tra khi làm” là công việc thực thi, **không phải đề nghị nhóm bỏ phiếu**. Nếu thay đổi hợp đồng, sửa file này và đồng bộ tài liệu của các module chịu ảnh hưởng.

| ID | Quyết định hoặc công việc liên quan | Trạng thái | Bước tiếp theo |
|---|---|---|---|
| D01 | `BAAI/bge-m3`, `FlagEmbedding`, dense 1024, L2-normalize, `float32`; dùng chung với HyDE | ✅ Đã chốt cấu hình | BƯỚC 01: pin revision/phiên bản thư viện thực tế; BƯỚC 08/10: kiểm thử |
| D02 | `wiki_en_<page_id>`, `chunk_id` có `_v1_`; Lucene ID = chunk ID; Qdrant UUIDv5 | ✅ Đã chốt quy ước | BƯỚC 05–09: tạo dữ liệu và xác thực mapping |
| D03 | Qdrant `wikipedia_chunks_v1`, Cosine, 1024 chiều, payload đã liệt kê | ✅ Đã chốt cấu hình | BƯỚC 08: tạo collection và kiểm thử |
| D04 | Corpus Wikipedia tiếng Anh, Plants and Animals; NaturalQuestionsV2 tùy chọn | ✅ Đã chốt phạm vi | BƯỚC 03: chọn cách thu thập cụ thể dựa trên dữ liệu nguồn |
| D05 | Tách OK-VQA làm dữ liệu đánh giá, không đưa vào corpus | ✅ Đã chốt nguyên tắc | BƯỚC 02: kiểm tra metadata thực tế rồi xác định cách lọc |
| D06 | PostgreSQL/Qdrant chạy local Docker Compose; Lucene chia sẻ bằng artifact/volume | ✅ Đã chốt cách triển khai ban đầu | BƯỚC 01 và BƯỚC 10: cấu hình, thử kết nối |
| D07 | Làm mẫu nhỏ → khoảng 5.000 chunks → cân nhắc 20.000–50.000 | ✅ Đã chốt định hướng | BƯỚC 11: kiểm tra chất lượng trước khi mở rộng |

**Mẫu ghi nhật ký sau mỗi phiên làm việc:**

```text
Ngày:
Bước đang làm:
Đã hoàn thành:
Minh chứng (file/commit/test):
Vướng mắc:
Quyết định cần cập nhật (nếu có):
Bước nhỏ tiếp theo:
```

## 7. Bản đồ phối hợp toàn dự án (ngoài phạm vi code chính của Quân)

Sau khi Data Indexing có dữ liệu dùng được, toàn nhóm còn cần:

- [ ] **HyDE Server:** Nhận `Image + Question`, sinh `P_hypo`, mã hóa thành `V_query` tương thích với Dense Index.
- [ ] **Retrieval Service:** Nhận `P_hypo` và `V_query`; BM25 + Dense Retrieval → RRF (đề xuất top 20) → Reranking → top 5 passages.
- [ ] **VQA Server:** Nhận `Image + Question + passages` rồi sinh câu trả lời ngắn.
- [ ] **Pipeline Integration:** Gọi tuần tự các HTTP API để chạy một câu hỏi và chạy theo lô.
- [ ] **Evaluation:** So sánh VQA Accuracy giữa baseline và các biến thể HyDE/Hybrid/Reranker; dùng các chỉ số Retrieval phù hợp với loại nhãn đang có.

Các thành viên HyDE/Retrieval/Evaluation **triển khai theo hợp đồng kỹ thuật Quân đã chốt ở BƯỚC 00** và phối hợp kiểm thử khi tích hợp; Quân không mặc định phải code thay các module đó.

## 8. Lưu ý kỹ thuật và nguyên tắc làm việc

1. **Phân biệt nguồn đánh giá và nguồn tri thức:** OK-VQA cấp ảnh, câu hỏi, đáp án để đánh giá; Wikipedia cấp passages để lập chỉ mục.
2. **ID là hợp đồng quan trọng nhất:** Sai mapping giữa ba nơi lưu trữ sẽ khiến hệ thống truy xuất sai đoạn văn dù BM25 và Dense Search đều chạy.
3. **Cùng encoder/cấu hình:** Dense document embeddings và HyDE query embeddings phải tương thích. Không chỉ kiểm tra có cùng số chiều.
4. **Không đánh đồng chứa đáp án với Recall@K:** Answer-Containment@K có thể dùng như chỉ số phụ; Recall@K/MRR đúng nghĩa cần nhãn passage liên quan phù hợp.
5. **Bắt đầu nhỏ, kiểm thử sớm:** Số bài mẫu và bước mở rộng trong kế hoạch này là đề xuất tổ chức thực hiện; các quy mô 5.000 và 20.000–50.000 chunks là định hướng của tài liệu dự án.
6. **Bảo mật và tái lập:** Không commit secret; ghi rõ phiên bản dữ liệu/model và cách chạy lại.
7. **Quy tắc hỗ trợ code:** Chỉ viết code khi Quân yêu cầu. Khi thiếu quyết định thiết kế quan trọng, cần hỏi/xác nhận trước; không tự giả định rồi code toàn bộ và đóng ZIP.

---

## 9. Công việc nên làm đầu tiên

**BƯỚC 00 đã chốt thiết kế. Bắt đầu thực hiện BƯỚC 01 — Chuẩn bị cấu trúc dự án và môi trường.**

- [x] Có hợp đồng kỹ thuật BGE-M3, PostgreSQL, BM25/Lucene và Qdrant tại mục BƯỚC 00.
- [x] Có schema tối thiểu và quy ước ID, ánh xạ giữa ba hệ thống.
- [x] Có lựa chọn Docker Compose local và cách Retrieval Service truy cập dữ liệu.
- [ ] Tạo/cập nhật thư mục `data-indexing/`, môi trường Python, Git, `.env.example`.
- [ ] Pin revision thực tế của model cùng phiên bản các dependencies; ghi rõ để thành viên HyDE dùng chung.
- [ ] Đưa hợp đồng kỹ thuật vào `docs/architecture.md` và các thông tin API liên quan vào `docs/api.md`.

**Lưu ý:** Chưa triển khai BƯỚC 01–12 chỉ vì thông số đã được chốt; hãy theo checklist và lưu minh chứng cụ thể.

**Nguồn xây dựng kế hoạch:** `dự-án.txt` (đặc biệt: phần “Thành phần 1 — Data Indexing”, tổ chức monorepo, hạ tầng, bàn giao và nguyên tắc phối hợp) và `hướng-dẫn-chat-gpt-viết-code.txt`. Các đầu việc như chạy thử bằng mẫu nhỏ, nhật ký tiến độ và tiêu chí nghiệm thu là phần **đề xuất chi tiết hóa** để dễ thực hiện từng bước, không phải yêu cầu mới đã được nhóm thống nhất.
