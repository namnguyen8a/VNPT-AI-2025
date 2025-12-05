### I. PHÂN TÍCH CHI TIẾT CUỘC THI

#### 1. Bài toán cốt lõi
Bạn không train lại model (finetuning), mà phải xây dựng một **Pipeline (Hệ thống xử lý)** sử dụng API của BTC để trả lời câu hỏi trắc nghiệm (A/B/C/D).
*   **Input:** Câu hỏi trắc nghiệm.
*   **Process:** Hệ thống Agent/RAG tìm kiếm thông tin -> Suy luận -> Chọn đáp án.
*   **Output:** Đáp án (A, B, C, hoặc D).

#### 2. Thách thức lớn nhất (Pain Points)
1.  **Hạn chế Tài nguyên:** Bạn **KHÔNG** được dùng model ngoài (GPT-4, Claude...) hay model local (Llama, Bert...) để inference hay embedding. Bắt buộc dùng 3 API của VNPT.
2.  **Dữ liệu:** BTC chỉ cho câu hỏi, **không cho tài liệu tham khảo (knowledge base)**. Bạn phải tự đi "cào" (crawl) dữ liệu từ internet để xây dựng Vector Database.
3.  **Đa dạng câu hỏi:** Có 5 loại câu hỏi đòi hỏi chiến lược xử lý khác nhau:
    *   *Bắt buộc không trả lời:* (Safety/Jailbreak detection).
    *   *Kiến thức chính xác:* (Cần RAG tốt - Lịch sử, Địa lý...).
    *   *Đọc hiểu văn bản dài:* (Cần Context Window management).
    *   *Toán học/Logic:* (LLM thường yếu cái này, cần Chain-of-Thought).
    *   *Đa lĩnh vực:* (Cần Router để phân loại).

#### 3. Tài nguyên BTC cấp (Lưu ý kỹ)
*   **LLM Mini:** Tốc độ nhanh, dùng để phân loại câu hỏi, tóm tắt, xử lý đơn giản (2000 req/ngày).
*   **LLM Large:** Thông minh hơn, dùng cho khâu suy luận cuối cùng (Reasoning) (1000 req/ngày).
*   **Embedding:** Dùng để mã hóa dữ liệu bạn thu thập được (Theo Q&A: 500 req/phút -> đủ để build data lớn, không lo giới hạn ngày).

---

### II. CHIẾN LƯỢC THỰC THI (FOLLOW THEO TIMELINE)

Cuộc thi diễn ra ngắn (12 ngày). Bạn cần chia giai đoạn như sau:

#### Giai đoạn 1: Chuẩn bị dữ liệu & Hạ tầng (Ngày 1 - Ngày 3)
*Mục tiêu: Có một kho tri thức (Knowledge Base) chất lượng.*

1.  **Thu thập dữ liệu (Crawling):** Đây là yếu tố **QUYẾT ĐỊNH THẮNG BẠI**. Vì mọi đội đều dùng chung model, đội nào có dữ liệu tham khảo tốt hơn sẽ thắng.
    *   *Nguồn:* Wikipedia Tiếng Việt (dump toàn bộ về), các trang Lịch sử Việt Nam, Địa lý, Văn bản pháp luật (thư viện pháp luật), Sách giáo khoa (nếu tìm được file text).
    *   *Xử lý:* Clean text, chia nhỏ (chunking) dữ liệu.
2.  **Xây dựng Vector Database:**
    *   Dùng API `vnptaiio_embedding` để vector hóa các chunk dữ liệu trên.
    *   Lưu vào ChromaDB hoặc FAISS (chạy local trong docker).
    *   **Lưu ý:** Tối ưu chunk size (khoảng 256-512 tokens) để tìm kiếm chính xác.

#### Giai đoạn 2: Xây dựng Pipeline Agent (Ngày 4 - Ngày 8)
*Mục tiêu: Code luồng xử lý cho từng loại câu hỏi.*

Bạn cần thiết kế một hệ thống **Router** (Phân luồng) sử dụng `LLM Mini`:
*   **Bước 1: Phân loại câu hỏi.** Prompt LLM Mini để xác định câu hỏi thuộc nhóm nào (An toàn, Kiến thức, Toán, Đọc hiểu).
*   **Bước 2: Xử lý chuyên biệt:**
    *   **Nhóm An toàn:** Nếu câu hỏi vi phạm chính trị/văn hóa -> Output theo quy định (ví dụ: chọn đáp án từ chối hoặc output rỗng tùy rule đề bài).
    *   **Nhóm Kiến thức (Sử/Địa/Văn):** Query vào Vector DB -> Lấy Top-K đoạn văn bản liên quan -> Đưa vào Prompt của `LLM Large` -> Trả lời.
    *   **Nhóm Toán/Logic:** Sử dụng kỹ thuật **Chain-of-Thought (CoT)**. Prompt: *"Hãy suy nghĩ từng bước một trước khi đưa ra đáp án"* (Let's think step by step).
    *   **Nhóm Đọc hiểu:** Đưa đoạn văn vào prompt (cẩn thận độ dài context) -> Hỏi.

#### Giai đoạn 3: Tối ưu & Evaluation (Ngày 9 - Ngày 11)
*Mục tiêu: Tăng độ chính xác và tốc độ.*

1.  **Chạy trên Dev Set (100 câu):**
    *   Đo đạc độ chính xác.
    *   Phân tích các câu sai (Error Analysis): Sai do không tìm thấy dữ liệu? Hay do LLM suy luận sai?
    *   Nếu không tìm thấy dữ liệu -> Crawl thêm dữ liệu mảng đó.
    *   Nếu LLM sai -> Tối ưu Prompt (Prompt Engineering).
2.  **Caching (Rất quan trọng):**
    *   Cache lại kết quả của các câu hỏi đã hỏi. Nếu gặp câu tương tự thì trả lời luôn, không gọi API để tiết kiệm quota và thời gian (Time-to-first-token).
    *   Cache lại các vector embedding để không phải embed lại khi chạy docker chấm điểm.

#### Giai đoạn 4: Đóng gói & Submit (Ngày 12)
1.  **Dockerize:** Đóng gói code, file requirements.txt, và **quan trọng nhất là file Vector Database** (đã build xong) vào Docker image.
2.  **Kiểm tra entry-point:** Đảm bảo script đọc đúng file `/data/public_test.csv` và ghi ra `/output/pred.csv`.
3.  **Viết báo cáo:** Nhấn mạnh vào: Chiến lược thu thập dữ liệu, cách build Router, và các kỹ thuật Prompting.

---

### III. CÁC MẸO "THỰC CHIẾN" (TIPS & TRICKS)

1.  **Hack Quota:**
    *   Vì Embedding có giới hạn/phút nhưng nới lỏng tổng/ngày sau khai mạc: Hãy viết script chạy đa luồng (multi-thread) nhưng có `sleep` để không vượt quá 500 req/phút, chạy liên tục 24/7 trong mấy ngày đầu để embed càng nhiều dữ liệu càng tốt.
2.  **Prompt Engineering:**
    *   Luôn dùng **Few-shot prompting**: Cung cấp cho LLM 1-2 ví dụ mẫu trong prompt (lấy từ Dev set) để nó học cách trả lời đúng định dạng.
    *   Với câu hỏi tiếng Việt, hãy prompt bằng tiếng Việt rõ ràng, mạch lạc.
3.  **Xử lý Toán học:**
    *   Nếu được phép chạy code Python local (cần check kỹ lại, thường Agent được phép dùng tool code calculator): Hãy dùng LLM để viết code Python giải bài toán -> Chạy code lấy kết quả -> Trả lời. (Nếu không cho chạy code thì bắt buộc dùng CoT).
4.  **Chiến thuật dữ liệu (Keyword Search vs Vector Search):**
    *   Đôi khi Vector Search (tìm theo ngữ nghĩa) không tốt bằng Keyword Search (BM25) với các từ khóa tên riêng, địa danh. Nên kết hợp **Hybrid Search** (dùng thư viện như BM25 kết hợp ChromaDB) để tăng độ chính xác khi retrieve dữ liệu.
5.  **Bẫy "Không sử dụng model ngoài":**
    *   Tuyệt đối không để sót bất kỳ dòng code nào gọi OpenAI hay Google trong code nộp.
    *   Tuy nhiên, trong quá trình *chuẩn bị dữ liệu* ở local (trước khi build docker), bạn có thể dùng các model mạnh hơn để clean data hoặc generate giả lập câu hỏi để test (Synthetic Data), miễn là trong Docker nộp đi không chứa code gọi model ngoài.

### IV. VIỆC CẦN LÀM NGAY HÔM NAY (05/12)

1.  Đăng ký API key và test thử kết nối ngay lập tức.
2.  Viết script Python đơn giản để gọi thử 3 API xem latency và format trả về.
3.  Lên danh sách các nguồn dữ liệu cần crawl (Wikipedia Vietnam là ưu tiên số 1).
4.  Setup Git repo và cấu trúc thư mục dự án.