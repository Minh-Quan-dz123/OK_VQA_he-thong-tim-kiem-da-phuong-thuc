# Đề tài : OK-VQA: A Visual Question Answering Benchmark Requiring External Knowledge là bài toán Knowledge-Based Visual

## flow

### Giao đoạn 0: Lập chỉ mục Offline cho cơ sở tri thức wikipedia
**Mục tiêu:** chuẩn bị CSDL để hỗ trợ Tìm Kiếm Lai (Hydrid Search) kết hợp cả so khớp từ khóa (Sparse) và ngữ nghĩa (Dense)  
**Input:** Tập hợp N đoạn văn Wikipedia (P1, P2, ..., P_N) thuộc miền Plants and Animals. Có thể bổ sung tài liệu Wikipedia phù hợp từ NaturalQuestionsV2 khi cần. Chunking mục tiêu khoảng 200 từ/chunk, overlap 40 từ. **Ảnh, câu hỏi và đáp án của OK-VQA chỉ phục vụ đánh giá; không đưa câu hỏi/đáp án OK-VQA vào corpus truy xuất.**  
Xử lý:   
- B1.1. đưa N đoạn văn vào bộ lập chỉ mục BM25 (như Pyserini / Lucene) để xây dựng chỉ mục đảo (Inverted Index) (thêm so với Base)
- B1.2. (song song) Dense indexing: đưa từng đoan văn P_i qua Text Embedding Model như BGE - M3 hoặc E5 - Large - v2 (kiến trúc Bi-encoder) để tạo vector ngữ nghĩa dày đặc Vp_i Rd (d=1024).  => sau đó khi toàn bộ Vp_i vào Vector Database (FAISS / Qdrant / Milvus) và xây dựng chỉ mục HNSW/IVF để tìm kiếm nhanh
**Output**  
Inverted Index (Sparse DB) phục vụ tìm kiếm từ khóa BM25 (thêm so với Base)  
Vector Database (Dense DB) lưu N vector {Vp_1, Vp_2,..., Vp_N}  

### Giai đoạn 1 : Biến đổi truy vấn Đa phương thức bằng HyDE (Multimodal HyDE Query Transformation) 
**Mục tiêu:** Giải quyết sự bất đồng không gian giữa (Ảnh + Câu hỏi) và Đoạn văn Wikipedia bằng kỹ thuật HyDE (Hypothetical Document Embeddings)   
**Input:** [Ảnh I,  câu hỏi Q] lấy từ OK - VQA  
xử lý:   
- B1. Sinh đoạn văn giả định: Đưa {I, Q} vào mô hình Vison - Language Model (VLM) như LLaVA - 1.5 - 7B hoặc Qwen - VL - Chat để tự động sinh ra một câu trả lời / đoạn văn giải thích, gọi là giả định P_hypo dưới dạng văn bản

Phypo=VLM(I,Q)  
- B2. Mã hóa truy vấn: Đưa chuỗi văn bản P_hypo qua mô hình Text Encoder (dùng cùng loại BGE - M3 hoặc E5 - large -v2 ở giai đoạn 0) để trích xuất vector truy vấn V_query Rd:  
Vquery=TextEncoder(Phypo)  
(so với Base thì V_quey = fusion(I, Q))  
**Output:** gồm 2 cái  
1 là chuỗi văn bản giả định Phypo (dùng cho Sparse Search). (có thêm so với Base)  
2 là Vector truy vấn V_query (dùng cho Dense Search)  

### Giai đoạn 2. Truy xuất tri thức Lai và Tái xếp hạng (Hybrid Search & Reranking) 
**Mục tiêu:** Kết hợp ưu điểm của BM25 và Dense Search để tăng Recall, sau đó dùng Reranker để tối ưu Precision  
**Input:**   
{P_hypo; V_query} lấy từ giai đoạn 1  
{Sparse DB & Vector DB} từ giai đoạn 0, tham số K1=50 (truy xuất thô), k2 = 5 (sau Rerack) (K1,K2 có thể thay đổi)  
**Xử lý**  
- B1. Retrieval song song
Sparse Retrieval (MB25): Dùng P_hypo tìm kiếm trên Inverted Index => Lấy được Top - K1 đoạn văn L_sparse cùng thứ hạng đi kèm ký hiệu là r_sparse(d)  
Dense Retrieval: Tính độ tương đồng Cosine giữa V_query và từng Vp_i trong Vector DB => lấy ra Top - K1 đoạn văn L_dense cùng thứ hạng đi kèm ký hiệu là r_dense(d)  
(L_sparse và L_dense là ký hiệu cho tập hợp {P_1,P_2....P_n} )  

- B2. Hợp nhất 2 kết quả trên bằng Reciprocal Rank Fusion RRF
Tính điểm tổng hợp cho mỗi đoạn văn dLsparseLdense bằng công thức :  

rồi sau đó lấy Top - 20 đoạn văn có S_RRF(d) cao nhất, ký hiệu là L_fused (là tập 20 P_i ổn áp nhất)  
- B3. Tái xếp hạng (Reranking)
Đưa từng cặp (P_hypo, P_i) trong L_fused qua mô hình Cross - Encoder Reranker (Như BGE - Reranker - large hoặc mMARCO Cross - Encoder) để tính điểm liên quan ngữ nghĩa chính xác Score_rerank(P_hypo, P_i)  
sau đó sắp xếp giảm dần theo Score_rerank và chọn lọc ra Top - K2 (K2=5) đoạn văn tốt nhất  
**Output**  
Danh sách Top - K đoạn văn wikipedia  chất lượng cao P_topK = { P_1, P_2,....P_K} (output thì giống hệt Base)  

### Giai đoạn 3 : Suy luận và sinh câu trả lời
Kết hợp thông tin hình ảnh I, câu hỏi Q và  P_topK = { P_1, P_2,....P_K} để sinh đáp án chính xác  
**Input:**  
{I, Q, Top - K đoạn văn lấy ra từ giai đoạn 2 (chính là P_topK)}  
**Xử lý:**  
- B1. Đóng gói ngữ cảnh (Prompt Construcion): Ghép Q và Top - K đoạn văn đó thành một chuỗi văn bản đầu vào Text Input = Q+"Context:” +P(1)​+P(2)​+⋯+P(K2)​ + “Based on the image and context, answer the question in 1 to 3 words.” 
- B2. Suy luận VLM: Đưa I và Text Input vào mô hình VQA như LLaVA - 1.5 - 13B, Qwen - VL - Chat, hoặc instructBLIP 
**Output:** Câu trả lời dự đoán A_pred (là chuỗi văn bản đáp án)  
### Giai đoạn 4: Đánh giá Hiệu năng Hệ thống (Evaluation Phase) 
Đối chiếu kết quả sinh ra với tập đáp án chuẩn của OK-VQA.   
**Input:**   
A_pred (từ Giai đoạn 3)  
tập 10, 15 hoặc N đáp án do con người gán nhãn trong OK-VQA cho câu hỏi tương ứng: A_gt = {a1, a2, … a10}  
Danh sách Top - K2 đoạn văn P_topK(từ giai đoạn 2)  
xử lý  
đánh giá module VQA theo tính điểm theo công thức  
VQAAccuracy(Apred)=1,SốđápántrongAgtkhớpvớiApred3  
Đánh giá module truy vấn (IR Evaluation): tính chỉ số Recal@K hoặc MRR để kiểm tra xem trong Top - K đoạn văn lấy ra ở Giai đoạn 2 có chứa đáp án đúng thuộc A_gt hay ko  
**Output:**  
Điểm số VQA Accuracy (đánh giá tổng thể hệ thống).  
Điểm số Recall@K / MRR (đánh giá riêng hiệu quả của module truy xuất vector).  
### Triển khai
- Miền tri thức: Plants and Animals, sử dụng tiếng Anh.
- Nguồn đánh giá: OK-VQA, chỉ chọn các câu hỏi thuộc miền này. Tập OK-VQA gốc có 9.009 câu train và 5.046 câu validation trước khi lọc.
- Nguồn tri thức: Ưu tiên thu thập bài viết Wikipedia theo danh mục động vật và thực vật. NaturalQuestionsV2 là nguồn bổ sung, không bắt buộc tải toàn bộ ~183 GB dung lượng đĩa ước tính.
- Chunking: Mỗi chunk mục tiêu 200 từ, có overlap 40 từ. Bắt đầu với khoảng 5.000 chunks để kiểm thử, sau đó mở rộng đến 20.000–50.000 chunks nếu dữ liệu thu thập được phù hợp.
- Mô hình đề xuất: BGE-M3 (embedding 1.024 chiều), BGE-Reranker-Base và Qwen2.5-VL-3B-Instruct cho nguyên mẫu. Các mô hình này đều có tài liệu triển khai bằng Python.
- Nguyên tắc phối hợp: Tất cả dịch vụ thống nhất cấu trúc dữ liệu và API trước khi viết logic. Phần VLM có thể thay bằng mock để những người không có GPU vẫn làm việc được.

# KẾ HOẠCH TRIỂN KHAI HỆ THỐNG MULTIMODAL HYBRID RETRIEVAL

> **Trạng thái tài liệu:** Bản kế hoạch và dự thảo hợp đồng tích hợp kỹ thuật. Những mục được ghi **“Đề xuất — chờ nhóm xác nhận”** chưa phải là quyết định chung. Xem **Mục VI** để chốt Bước 00 của phần Data Indexing. Giữ nguyên những nội dung đã quy định trong dự án; không suy diễn rằng các công cụ đã cài đặt hoặc các module đã hoàn thành.
## I. Tổng quan
### 1. Mục tiêu
Xây dựng hệ thống Knowledge-Based Visual Question Answering (KB-VQA), sử dụng Multimodal HyDE kết hợp Hybrid Search (BM25 + Dense Retrieval), RRF và Reranking để tìm kiếm tri thức Wikipedia, sau đó dùng VLM sinh câu trả lời.  
Miền dữ liệu lựa chọn: Plants and Animals (Động vật và thực vật).  
**Nguồn dữ liệu:**  
- OK-VQA: Cung cấp hình ảnh, câu hỏi và đáp án chuẩn. Chỉ sử dụng các câu hỏi thuộc nhóm Plants and Animals.
- Wikipedia: Cung cấp các đoạn văn tri thức về động vật và thực vật.
- NaturalQuestionsV2: Có thể sử dụng để bổ sung các tài liệu Wikipedia phù hợp nếu cần.
Do NaturalQuestionsV2 có dung lượng rất lớn, hệ thống không cần tải hoặc lập chỉ mục toàn bộ dataset. Chỉ cần xây dựng một corpus Wikipedia thuộc miền đã chọn.  
Quy mô ban đầu dự kiến khoảng 5.000–20.000 chunks, có thể mở rộng sau khi kiểm tra chất lượng dữ liệu.  
### 2. Công nghệ
- Ngôn ngữ: Python.
- Backend API: FastAPI.
- Deep Learning: PyTorch, Transformers.
- VLM: Qwen-VL / LLaVA hoặc mô hình phù hợp.
- Text Embedding: BGE-M3 (ưu tiên) hoặc E5.
- Sparse Retrieval: Pyserini / Lucene (BM25).
- Dense Retrieval: Qdrant.
- Reranker: BGE-Reranker.
- Database: PostgreSQL.
Model cụ thể, phiên bản thư viện và cấu hình phần cứng do thành viên phụ trách đề xuất, sau đó cả nhóm thống nhất để bảo đảm các module tương thích.  

## II. Kiến trúc hệ thống
Hệ thống gồm 5 thành phần chính và 1 thành phần tích hợp.  
### Giai đoạn 0 — Data Indexing (Offline)
```text
Wikipedia (Plants and Animals)
              |
              v
       Preprocessing
              |
              v
     Chunking ~200 words
              |
              v
         PostgreSQL
              |
       +------+------+
       |             |
       v             v
   BM25 Index    Dense Index
    (Lucene)      (Qdrant)
```

### Giai đoạn 1–4 — Online Pipeline
```text
Image + Question
        |
        v
    HyDE Server
        |
  +-----+-----+
  |           |
P_hypo      V_query
  |           |
  v           v
 BM25       Dense
  |           |
  +-----+-----+
        |
        v
       RRF
        |
        v
    Reranking
        |
        v
   Top-K Passages
        |
        v
    VQA Server
        |
        v
      Answer
        |
        v
    Evaluation
```

Các thành phần server hoạt động độc lập, giao tiếp thông qua HTTP API.  

## III. Tổ chức Monorepo
Cấu trúc dự án dự kiến:  
```text
multimodal-hybrid-retrieval/
│
├── data-indexing/
│   ├── src/
│   ├── tests/
│   ├── README.md
│   ├── requirements.txt
│   └── .env.example
│
├── hyde-server/
│   ├── app/
│   ├── tests/
│   ├── README.md
│   ├── requirements.txt
│   └── .env.example
│
├── retrieval-service/
│   ├── app/
│   ├── tests/
│   ├── README.md
│   ├── requirements.txt
│   └── .env.example
│
├── vqa-server/
│   ├── app/
│   ├── tests/
│   ├── README.md
│   ├── requirements.txt
│   └── .env.example
│
├── evaluation/
│   ├── src/
│   ├── results/
│   └── README.md
│
├── pipeline/
│   ├── run_pipeline.py
│   └── README.md
│
├── data/
│   ├── raw/
│   ├── processed/
│   └── indexes/
│
├── docs/
│   ├── architecture.md
│   ├── api.md
│   └── setup.md
│
├── docker/
│
├── docker-compose.yml
├── .gitignore
└── README.md
```

Mỗi thành viên quản lý một thư mục độc lập. Các module có môi trường Python riêng để tránh xung đột dependencies.  
README của từng thành phần phải giải thích cách cài đặt, cấu hình, chạy và kiểm thử.  

## IV. Phân chia thành phần
### Thành phần 1 — Data Indexing
**Người phụ trách:** _____ Quân_____  
**Giai đoạn:** 0  
**Loại:** Các chương trình Python xử lý Offline, không phải server.  
#### 1. Nhiệm vụ
Chuẩn bị dữ liệu Wikipedia thuộc miền Plants and Animals và xây dựng hai hệ thống chỉ mục phục vụ Hybrid Search.  
- Các công việc chính:
- Chuẩn bị dữ liệu OK-VQA, lọc các câu hỏi Plants and Animals để phục vụ đánh giá.
- Thu thập Wikipedia về động vật và thực vật, có thể kết hợp NaturalQuestionsV2 nếu cần.
- Làm sạch, loại bỏ trùng lặp và chia tài liệu thành chunks khoảng 200 từ.
- Lưu nội dung chunks và metadata vào PostgreSQL.
- Xây dựng Sparse Index bằng Pyserini/Lucene.
- Sử dụng Text Encoder tạo embeddings và lưu vào Qdrant.
Lưu ý: Câu hỏi và đáp án của OK-VQA không được đưa vào Wikipedia Corpus dùng cho Retrieval.  
#### 2. Các file dự kiến
```text
data-indexing/
│
├── src/
│   ├── download_okvqa.py
│   ├── collect_wikipedia.py
│   ├── preprocess.py
│   ├── chunking.py
│   ├── database.py
│   ├── build_sparse_index.py
│   ├── build_dense_index.py
│   └── validate_indexes.py
│
├── tests/
├── README.md
├── requirements.txt
└── .env.example
```

**Mô tả:**  
- download_okvqa.py: Tải và lọc OK-VQA.
- collect_wikipedia.py: Thu thập các bài viết Wikipedia phù hợp với miền dữ liệu.
- preprocess.py: Làm sạch và loại bỏ dữ liệu trùng lặp.
- chunking.py: Chia tài liệu thành chunks khoảng 200 từ, overlap tùy cấu hình.
- database.py: Kết nối PostgreSQL, lưu documents và chunks.
- build_sparse_index.py: Xây dựng BM25 Index.
- build_dense_index.py: Tạo embedding và lưu vào Qdrant.
- validate_indexes.py: Kiểm tra dữ liệu và khả năng truy xuất từ hai index.
Có thể bổ sung hoặc gộp file nếu hợp lý.  
#### 3. Cấu trúc dữ liệu
**PostgreSQL cần lưu ít nhất:**  
**Documents:**  
```text
document_id
title
source
url
domain
```

**Chunks:**  
```text
chunk_id
document_id
text
chunk_index
```

Các trường bổ sung do người phụ trách quyết định.  
Một `chunk_id` phải ánh xạ nhất quán giữa PostgreSQL, Lucene và Qdrant để Retrieval Service lấy đúng nội dung đoạn văn. Quy ước chi tiết về ID, cách lưu ở từng hệ thống và cấu hình kết nối xem **Mục VI**.  
> **Lưu ý kỹ thuật:** Qdrant chỉ chấp nhận point ID ở dạng số nguyên không âm hoặc UUID; vì vậy không mặc định dùng trực tiếp chuỗi `chunk_id` làm point ID. Có thể lưu `chunk_id` trong payload và dùng UUID xác định cho point ID (đang đề xuất, chờ thống nhất).  
#### 4. Hạ tầng
- PostgreSQL: Lưu nội dung và metadata.
- Lucene: Lưu Sparse Index trên ổ đĩa.
- Qdrant: Lưu Dense Vectors.
Có thể triển khai PostgreSQL và Qdrant bằng Docker trên máy local hoặc sử dụng cloud.  
#### 5. Kết quả bàn giao
- Corpus đã xử lý.
- PostgreSQL chứa documents/chunks.
- Lucene Index.
- Qdrant Collection.
- Các script xây dựng lại dữ liệu.
- README hướng dẫn cài đặt và sử dụng các hệ thống lưu trữ.
- Dữ liệu mẫu để thành viên Retrieval Service kiểm thử.

### Thành phần 2 — HyDE Server
**Người phụ trách:** ______ ________  
**Giai đoạn:** 1  
**Loại:** FastAPI Server.  
#### 1. Nhiệm vụ
Nhận Image + Question, sử dụng VLM sinh đoạn văn giả định P_hypo. Sau đó dùng Text Encoder chuyển P_hypo thành Query Vector.  
```text
Image + Question
        |
        v
       VLM
        |
        v
      P_hypo
        |
        v
   Text Encoder
        |
        v
      V_query
```

Server không cần kết nối database.  
#### 2. Các file dự kiến
```text
hyde-server/
│
├── app/
│   ├── main.py
│   ├── schemas.py
│   ├── vlm.py
│   ├── embedding.py
│   └── service.py
│
├── tests/
├── README.md
├── requirements.txt
└── .env.example
```

**Nhiệm vụ:**  
- main.py: Khởi tạo server và khai báo API.
- schemas.py: Định nghĩa dữ liệu request/response.
- vlm.py: Load VLM và sinh Hypothetical Document.
- embedding.py: Load Text Encoder và tạo Query Vector.
- service.py: Điều phối quá trình xử lý.
#### 3. API
POST /api/v1/hyde  
**Request:** multipart/form-data  
image: Image file  
question: "What animal is this?"  
**Response:**  
```json
{
  "hypothetical_document": "The image shows a zebra...",
  "query_vector": [0.12, 0.83, -0.24]
}
```

Vector minh họa đã rút gọn. Số chiều thực tế phụ thuộc model đã thống nhất (BGE-M3 là 1.024).  
#### 4. Kết quả bàn giao
- Server chạy được độc lập.
- API nhận ảnh và câu hỏi.
- Sinh P_hypo và V_query.
- README hướng dẫn cài đặt, chạy và test API.
Lưu ý: Text Encoder phải tương thích với model đã sử dụng để lập Dense Index tại Thành phần 1. Cần chốt cùng checkpoint/revision, thư viện suy luận, cách encode và các cấu hình ảnh hưởng trực tiếp đến vector; xem **Mục VI.2**.  

### Thành phần 3 — Retrieval Service
**Người phụ trách:** _____ ______  
**Giai đoạn:** 2  
**Loại:** FastAPI Server.  
#### 1. Nhiệm vụ
Nhận P_hypo và V_query từ HyDE Server, sau đó thực hiện Hybrid Retrieval:  
- BM25 Search trên Lucene Index, lấy Top-K1.
- Dense Search trên Qdrant, lấy Top-K1.
- Hợp nhất kết quả bằng RRF.
- Chọn Top-20 ứng viên.
- Reranking bằng Cross-Encoder.
- Trả về Top-K2 đoạn Wikipedia tốt nhất.
**Tham số ban đầu:**  
- K1 = 50
- RRF Top-K = 20
- K2 = 5
- RRF constant = 60
Các tham số có thể điều chỉnh thông qua cấu hình hoặc request.  
#### 2. Các file dự kiến
```text
retrieval-service/
│
├── app/
│   ├── main.py
│   ├── schemas.py
│   ├── database.py
│   ├── sparse.py
│   ├── dense.py
│   ├── fusion.py
│   ├── reranker.py
│   └── service.py
│
├── tests/
├── README.md
├── requirements.txt
└── .env.example
```

**Mô tả:**  
- database.py: Kết nối PostgreSQL và truy xuất chunks.
- sparse.py: BM25 Search.
- dense.py: Qdrant Vector Search.
- fusion.py: Hợp nhất kết quả bằng RRF.
- reranker.py: Reranking các đoạn văn.
- service.py: Điều phối toàn bộ quá trình Retrieval.
#### 3. API
POST /api/v1/retrieve  
**Request:**  
```json
{
  "hypothetical_document": "The image shows a zebra...",
  "query_vector": [0.12, 0.83, -0.24],
  "top_k_retrieval": 50,
  "top_k_rerank": 5
}
```

**Response:**  
```json
{
  "passages": [
    {
      "chunk_id": "chunk_001",
      "title": "Zebra",
      "text": "Zebras are African equines...",
      "score": 0.95
    },
    {
      "chunk_id": "chunk_002",
      "title": "Equidae",
      "text": "Equidae is the taxonomic family...",
      "score": 0.89
    }
  ]
}
```

Ví dụ response đã rút gọn, số đoạn thực tế tối đa bằng K2.  
#### 4. Kết quả bàn giao
- Server Retrieval hoạt động độc lập.
- Kết nối được PostgreSQL, Lucene và Qdrant theo hợp đồng truy cập ở **Mục VI.5**.
- Thực hiện BM25, Dense Search, RRF và Reranking.
- API trả danh sách passages theo format thống nhất.
- README hướng dẫn cấu hình, chạy và kiểm thử.

### Thành phần 4 — VQA Server
**Người phụ trách:** _______ Thu Hằng _____  
**Giai đoạn:** 3  
**Loại:** FastAPI Server.  
#### 1. Nhiệm vụ
Nhận Image + Question + Top-K Wikipedia Passages, đưa vào VLM để sinh câu trả lời cuối cùng.  
```text
Image + Question + Top-K Passages
                |
                v
          Prompt Builder
                |
                v
               VLM
                |
                v
              Answer
```

Server không kết nối đến PostgreSQL, Qdrant hoặc Lucene.  
#### 2. Các file dự kiến
```text
vqa-server/
│
├── app/
│   ├── main.py
│   ├── schemas.py
│   ├── vlm.py
│   ├── prompts.py
│   └── service.py
│
├── tests/
├── README.md
├── requirements.txt
└── .env.example
```

**Mô tả:**  
- main.py: FastAPI và route.
- schemas.py: Request/Response.
- vlm.py: Load model và suy luận.
- prompts.py: Xây dựng prompt từ Question và Contexts.
- service.py: Điều phối xử lý.
#### 3. API
POST /api/v1/vqa  
**Request:** multipart/form-data  
image: Image file  

question:  
"What is the closest relative of this animal?"  

contexts_json:  
```json
[
  {
    "chunk_id": "chunk_001",
    "text": "The closest living relative of giraffes is the okapi."
  }
]
```

**Response:**  
```json
{
  "answer": "okapi"
}
```

Yêu cầu VLM trả lời ngắn gọn, ưu tiên 1–3 từ, phù hợp với OK-VQA.  
#### 4. Kết quả bàn giao
- Server VQA chạy độc lập.
- Nhận Image, Question và Contexts.
- Tạo prompt và chạy VLM.
- Trả về Answer.
- Có README và test API.

### Thành phần 5 — Evaluation
**Người phụ trách:** ______ chưa có _____  
**Giai đoạn:** 4  
**Loại:** Offline Python Scripts, không phải server.  
#### 1. Nhiệm vụ
Đánh giá hiệu quả toàn bộ hệ thống và hiệu quả riêng của Retrieval.  
**VQA Evaluation:**  
- So sánh câu trả lời dự đoán A_pred với các đáp án A_gt trong OK-VQA.
- Sử dụng VQA Accuracy theo quy tắc đánh giá chuẩn.
**Retrieval Evaluation:**  
- Kiểm tra các passages truy xuất có chứa thông tin cần thiết để trả lời câu hỏi hay không.
- Đánh giá Hit@K, Recall@K, MRR nếu có relevance labels phù hợp.
- Có thể sử dụng Answer-Containment@K như chỉ số phụ.
- Lưu ý: Kiểm tra passage có chứa chuỗi đáp án không đồng nghĩa với Recall@K chuẩn. Nếu đánh giá Recall/MRR thì cần xây dựng tập nhãn passage liên quan cho một số câu hỏi.
#### 2. Các file dự kiến
```text
evaluation/
│
├── src/
│   ├── prepare_eval.py
│   ├── evaluate_vqa.py
│   ├── evaluate_retrieval.py
│   └── compare_results.py
│
├── results/
└── README.md
```

#### 3. Các thí nghiệm đề xuất
Thí nghiệm  
Phương pháp  
Base 1  
VLM không dùng Retrieval  
Base 2  
BM25 Retrieval  
Base 3  
Dense Retrieval  
Base 4  
Hybrid Retrieval + RRF  
Advanced 1  
Multimodal HyDE + Hybrid  
Advanced 2  
Multimodal HyDE + Hybrid + Reranker  

Có thể bổ sung thí nghiệm Hybrid + Reranker không dùng HyDE để đo riêng hiệu quả của HyDE.  
#### 4. Kết quả bàn giao
- Script đánh giá.
- Kết quả VQA Accuracy.
- Các chỉ số Retrieval.
- Bảng so sánh giữa các phương pháp.
- Báo cáo phân tích kết quả.

### Thành phần 6 — Pipeline Integration
**Người phụ trách:** _____ chưa có______  
#### 1. Nhiệm vụ
Viết chương trình Python gọi tuần tự các API để hoàn thành quy trình End-to-End.  
```text
Image + Question
      |
      v
POST /api/v1/hyde
      |
      v
POST /api/v1/retrieve
      |
      v
POST /api/v1/vqa
      |
      v
     Answer
```

#### 2. Các file dự kiến
```text
pipeline/
│
├── run_pipeline.py
├── run_batch.py
├── client.py
└── README.md
```

- client.py: Các hàm gọi API.
- run_pipeline.py: Chạy thử một ảnh và câu hỏi.
- run_batch.py: Chạy nhiều câu hỏi OK-VQA và lưu kết quả.
Không cần xây dựng thêm server cho Pipeline.  

## V. Quy định chung khi triển khai
### 1. Làm việc độc lập
Mỗi thành viên cần triển khai thành phần của mình sao cho có thể chạy và test riêng.  
Trong giai đoạn đầu, các thành viên được phép sử dụng dữ liệu giả lập (Mock Data) để kiểm tra API trước khi tích hợp model hoặc database thật.  
**Ví dụ:**  
- Retrieval Service có thể nhận P_hypo và vector mẫu mà không cần HyDE Server.
- VQA Server có thể nhận passages giả lập mà không cần Retrieval Service.
- Pipeline có thể sử dụng mock responses để kiểm tra quá trình giao tiếp.
### 2. Thống nhất API
Các thành viên phải thống nhất request và response format trước khi triển khai.  
Khi muốn thay đổi API phải cập nhật docs/api.md và thông báo cho các thành viên liên quan.  
### 3. Quản lý Git
**Sử dụng:**  
```text
main
dev
feature/data-indexing
feature/hyde-server
feature/retrieval-service
feature/vqa-server
feature/evaluation
```

Mỗi thành viên phát triển trên branch riêng, sau đó merge vào dev.  
### 4. Tài liệu bàn giao
**Mỗi thành phần phải có:**  
- README hướng dẫn cài đặt.
- Danh sách dependencies.
- .env.example.
- Hướng dẫn chạy chương trình hoặc server.
- Ví dụ input/output.
- Hướng dẫn test.
- Thông tin cấu hình model hoặc database.
Người khác phải có thể làm theo tài liệu để chạy được thành phần đó trên máy cá nhân hoặc cloud.
## VI. Hợp đồng kỹ thuật Data Indexing ↔ HyDE ↔ Retrieval (BƯỚC 00)

**Mục đích:** Làm rõ những thông số cần thống nhất **trước khi xây dựng Dense Index, Sparse Index và tích hợp server**. Phần này là **dự thảo để nhóm xem xét**, không tự động thay thế thỏa thuận của nhóm. Người phụ trách Data Indexing phối hợp với thành viên HyDE và Retrieval để cập nhật kết quả chốt vào `docs/architecture.md` và `docs/api.md`.

### VI.1. Quy định đã có trong kế hoạch

| Hạng mục | Quy định hiện có | Trạng thái |
|---|---|---|
| Ngôn ngữ / phạm vi | Wikipedia tiếng Anh, Plants and Animals | Đã nêu trong kế hoạch |
| Dữ liệu đánh giá | OK-VQA, lọc Plants and Animals | Đã nêu trong kế hoạch |
| Dữ liệu không được đưa vào corpus truy xuất | Câu hỏi và đáp án OK-VQA | Đã nêu trong kế hoạch |
| Phân đoạn | Mục tiêu ~200 từ/chunk, overlap 40 từ | Đã nêu trong kế hoạch |
| Dữ liệu gốc/metadata | PostgreSQL (`documents`, `chunks`) | Đã nêu trong kế hoạch |
| Sparse Retrieval | BM25 bằng Pyserini/Lucene | Đã nêu trong kế hoạch |
| Dense Retrieval | Qdrant | Đã nêu trong kế hoạch |
| Text Encoder | Ưu tiên BGE-M3 (1.024 chiều); có thể cân nhắc E5 | **Chưa chốt chính thức** |
| Phương pháp so sánh vector | Cosine được mô tả ở giai đoạn Retrieval | Đã nêu về thuật toán; **chưa chốt cấu hình Collection** |

### VI.2. Thống nhất Text Encoder giữa Data Indexing và HyDE

**Đề xuất — chờ nhóm xác nhận:**

| Thông số | Giá trị đề xuất / nội dung cần thống nhất |
|---|---|
| Model | `BAAI/bge-m3` |
| Output dùng để Dense Retrieval | Dense embedding (không dùng nhầm sparse vector hoặc multi-vector đầu ra khác) |
| Số chiều | `1024` nếu sử dụng BGE-M3 với cấu hình mặc định |
| Checkpoint | Ghi **revision/commit hash cụ thể** sau khi tải model; hiện **chưa chọn** |
| Thư viện tạo embedding | Chọn một cách encode nhất quán (ví dụ `FlagEmbedding` hoặc cách tích hợp đã kiểm chứng); **chưa chọn phiên bản** |
| Chuẩn bị văn bản | Ghi quy tắc làm sạch đầu vào, có/không thêm prefix, giới hạn độ dài/truncation |
| Chuẩn hóa / kiểu vector | Ghi rõ có normalize hay không, float32 hay kiểu khác; kiểm tra sự tương thích với metric Qdrant |
| Kiểm thử | Hai bên encode cùng một chuỗi thử và so sánh kích thước, độ chênh vector theo ngưỡng đã thống nhất |

**Nguyên tắc bắt buộc:** Embeddings của Wikipedia (Data Indexing) và embeddings của `P_hypo` (HyDE) phải được tạo trong cùng không gian vector. Chỉ ghi tên “BGE-M3” là **chưa đủ** nếu khác checkpoint, quy trình encode hoặc cấu hình gây thay đổi biểu diễn. Cách triển khai có thể khác máy nhưng phải được kiểm thử tương thích.

**Người xác nhận:** Data Indexing: ______ · HyDE: ______ · Ngày: ______

### VI.3. Quy ước `document_id`, `chunk_id` và đồng bộ ba hệ thống

**Schema tối thiểu đã có trong dự án:**

```text
documents(document_id, title, source, url, domain)
chunks(chunk_id, document_id, text, chunk_index)
```

**Đề xuất — chờ nhóm xác nhận:**

- `document_id`: chuỗi ổn định dựa trên ID trang Wikipedia tiếng Anh, ví dụ `wiki_en_12345`. Nếu dùng nguồn khác, phải quy định namespace riêng để không trùng ID.
- `chunk_index`: số nguyên bắt đầu từ `0` trong từng tài liệu.
- `chunk_id`: chuỗi duy nhất dạng `<document_id>_chunk_<chunk_index>`, ví dụ `wiki_en_12345_chunk_0000` (phần chỉ số đệm 4 chữ số). Khi thay đổi bản corpus hoặc thuật toán chunking, phải xác định cách version/rebuild để không khiến cùng ID trỏ sang nội dung khác.
- PostgreSQL: `documents.document_id` là khóa chính; `chunks.chunk_id` là khóa chính; `chunks.document_id` tham chiếu đến `documents.document_id`. Đề xuất thêm ràng buộc duy nhất cho cặp `(document_id, chunk_index)` trong từng phiên bản corpus.
- Lucene: sử dụng giá trị `chunk_id` làm định danh document trong chỉ mục để kết quả BM25 ánh xạ được về PostgreSQL.
- Qdrant: lưu `chunk_id` và `document_id` trong payload; dùng UUID ổn định (ví dụ UUIDv5 từ `chunk_id`) làm Qdrant point ID nếu cần. **Không dùng chuỗi `chunk_id` tùy ý làm point ID trực tiếp**.
- PostgreSQL lưu nội dung văn bản chuẩn; Lucene và Qdrant chứa dữ liệu đủ dùng để truy xuất và ánh xạ, không tự tạo định danh không thể đối chiếu.

**Kiểm thử bắt buộc:** Với một `chunk_id` mẫu, kiểm tra bản ghi PostgreSQL, kết quả BM25 và point Qdrant đều quy về **đúng cùng một đoạn văn**. Khi rebuild phải kiểm tra số lượng, tính duy nhất và phát hiện ID bị trỏ sang nội dung khác.

**Người xác nhận:** Data Indexing: ______ · Retrieval: ______ · Ngày: ______

### VI.4. Cấu hình Qdrant Collection

**Đề xuất — chờ nhóm xác nhận:**

| Thông số | Giá trị đề xuất |
|---|---|
| Collection | `wikipedia_chunks_v1` |
| Vector size | `1024` **nếu đã chốt BGE-M3** |
| Distance | `Cosine` |
| Point ID | UUID ổn định tạo từ `chunk_id`, hoặc quy tắc ID khác sau khi nhóm thống nhất |
| Payload bắt buộc | `chunk_id`, `document_id` |
| Payload bổ sung hữu ích | `domain`, `title`, `source` nếu Retrieval cần lọc/hiển thị nhanh |
| Nguồn nội dung chuẩn | PostgreSQL, truy vấn theo `chunk_id` |

**Điều kiện hoàn thành:** Collection được tạo đúng vector size/metric, có thể insert và tìm kiếm vector thử; Retrieval Service lấy đúng passage từ PostgreSQL qua `chunk_id`. Nếu đổi model hoặc kích thước vector thì tạo Collection phiên bản mới hoặc xây dựng lại chỉ mục, không ghép lẫn hai không gian vector.

**Người xác nhận:** Data Indexing: ______ · Retrieval: ______ · Ngày: ______

### VI.5. Cách Retrieval Service truy cập PostgreSQL, Lucene và Qdrant

**Hiện trạng:** Kế hoạch nêu Retrieval Service kết nối cả ba hệ thống, nhưng **chưa chốt cách triển khai chia sẻ giữa các máy**.

**Đề xuất — chờ nhóm xác nhận:**

| Tài nguyên | Cách Retrieval Service truy cập | Biến/cấu hình dự kiến |
|---|---|---|
| PostgreSQL | Kết nối theo URL và tài khoản được cấp; không hard-code mật khẩu | `DATABASE_URL` |
| Qdrant | Kết nối qua HTTP(S) endpoint; API key nếu môi trường yêu cầu | `QDRANT_URL`, `QDRANT_API_KEY`, `QDRANT_COLLECTION` |
| Lucene | Index là **thư mục trên đĩa**, không phải HTTP API mặc định. Cần bàn giao/copy index hoặc mount cùng volume cho Retrieval Service | `LUCENE_INDEX_PATH` |
| Bộ mã hóa | Đọc cấu hình model đã chốt để kiểm tra tương thích truy vấn | `EMBEDDING_MODEL_ID`, `EMBEDDING_MODEL_REVISION` (tên biến đề xuất) |

Cần thống nhất các trường hợp triển khai:

1. **Chạy cùng máy / cùng Docker Compose:** PostgreSQL và Qdrant là các service hoặc container; Lucene Index được mount vào Retrieval Service bằng volume phù hợp.
2. **Chạy trên các máy khác nhau:** PostgreSQL và Qdrant cần host/port hoặc endpoint có thể truy cập và được bảo vệ; Lucene Index phải được phân phối như một artifact có phiên bản và checksum, hoặc đặt trên storage được mount tương thích.
3. **Bí mật kết nối:** Đặt giá trị thực trong `.env` cục bộ hoặc kho secrets; chỉ đưa **tên biến và giá trị mẫu không nhạy cảm** vào `.env.example`. Không commit mật khẩu, token, dữ liệu lớn hoặc index khổng lồ lên Git.

**Thông tin phải bàn giao cho Retrieval Service:** tên Collection, model/revision và vector size, metric, quy ước ID, schema, đường dẫn hoặc artifact của Lucene Index, ví dụ `chunk_id` và câu truy vấn để test ba nguồn. Mọi thay đổi phải thông báo nhóm và cập nhật `docs/architecture.md` / `docs/api.md` khi ảnh hưởng hợp đồng tích hợp.

**Người xác nhận:** Data Indexing: ______ · Retrieval: ______ · Người phụ trách hạ tầng: ______ · Ngày: ______

### VI.6. Checklist ký xác nhận — điều kiện đóng BƯỚC 00

- [x] Ghi rõ phạm vi tiếng Anh, Plants and Animals và nguồn corpus/đánh giá theo kế hoạch hiện tại.
- [x] Ghi schema tối thiểu của `documents` và `chunks`.
- [x] Ghi cấu hình dự thảo và các vấn đề còn thiếu cần nhóm thảo luận.
- [ ] Data Indexing và HyDE chốt encoder, checkpoint/revision, cách tạo vector và test tương thích.
- [ ] Data Indexing và Retrieval chốt quy tắc ID, ánh xạ sang Lucene và Qdrant.
- [ ] Data Indexing và Retrieval chốt Qdrant Collection, vector size, metric và payload.
- [ ] Data Indexing và Retrieval chốt phương thức truy cập ba hệ thống, bao gồm bàn giao Lucene Index.
- [ ] Chuyển các quyết định **đã được nhóm xác nhận** vào `docs/architecture.md` và phần API liên quan vào `docs/api.md`.
- [ ] Ghi nhận ngày, người xác nhận và chuyển các mục “Đề xuất — chờ nhóm xác nhận” thành “Đã chốt”.

> **Kết luận trạng thái:** Đã **hoàn thành việc soạn dự thảo BƯỚC 00**, nhưng **chưa xác nhận BƯỚC 00 hoàn tất** cho đến khi các mục chưa đánh dấu ở trên được nhóm chốt. Trong khi chờ, có thể chuẩn bị cấu trúc dự án/môi trường và dữ liệu mẫu; chưa nên xây dựng Dense Index chính thức với thông số còn thay đổi.
