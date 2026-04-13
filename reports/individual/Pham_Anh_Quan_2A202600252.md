# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Phạm Anh Quân  
**Vai trò trong nhóm:** Tech Lead / Retrieval Owner  
**Ngày nộp:** 13/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

Với vai trò là Tech Lead và Retrieval Owner, tôi chịu trách nhiệm chính trong việc thiết kế kiến trúc tổng thể của hệ thống RAG và trực tiếp triển khai tầng truy xuất dữ liệu (Retrieval). Trong Sprint 1, tôi đã xây dựng quy trình Indexing dựa trên phương pháp **Hierarchical Chunking** (chia nhỏ tài liệu theo cấu trúc Section rồi đến Paragraph) để đảm bảo không làm mất ngữ cảnh của các điều khoản chính sách. 

Sang Sprint 2 và 3, tôi tập trung vào việc tối ưu hóa độ chính xác của tìm kiếm bằng cách triển khai **Hybrid Retrieval** (kết hợp Dense Search qua ChromaDB và Sparse Search qua BM25). Đặc biệt, tôi đã tích hợp thêm một bộ **Reranker (Cross-Encoder)** để chấm điểm lại các kết quả tìm kiếm thô. Công việc của tôi đóng vai trò là "màng lọc" quan trọng, cung cấp những mảnh ghép thông tin chuẩn xác nhất cho bộ phận Generation (LLM) để tạo ra câu trả lời cuối cùng. Tôi cũng hỗ trợ nhóm giải quyết các lỗi về cấp phát bộ nhớ (OOM) khi tải model local và cấu hình biến môi trường đa nền tảng.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

Khái niệm tôi tâm đắc nhất sau bài lab này là sự bổ trợ lẫn nhau giữa **Dense Retrieval** và **Sparse Retrieval**. Trước đây, tôi nghĩ chỉ cần Vector Search là đủ, nhưng thực tế cho thấy Vector Search (Dense) đôi khi rất kém trong việc tìm chính xác các từ khóa đặc định như mã lỗi (`ERR-403`) hay các mức độ ưu tiên (`P1`). Ngược lại, BM25 (Sparse) lại xử lý cực tốt các thuật ngữ này nhưng lại mù tịt về mặt ngữ nghĩa nếu người dùng dùng từ đồng nghĩa.

Ngoài ra, tôi cũng hiểu rõ hơn về cơ chế **Reciprocal Rank Fusion (RRF)**. Việc kết hợp hai luồng tìm kiếm không chỉ đơn giản là lấy kết quả của cả hai, mà cần một thuật toán chấm điểm lại thứ hạng để tìm ra điểm giao thoa tốt nhất. Cuối cùng, sự xuất hiện của **Reranker (Cross-Encoder)** như một "giám khảo" cuối cùng đã giúp tôi hiểu rằng một hệ thống RAG mạnh mẽ không chỉ cần tìm nhanh (bi-encoder) và còn cần sự thẩm định sâu (cross-encoder) trước khi gửi dữ liệu vào LLM.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

Điều khiến tôi ngạc nhiên nhất chính là **"Nghịch lý của sự phức tạp" (Complexity Paradox)**. Trong quá trình đánh giá A/B, tôi đã rất sốc khi thấy phiên bản Variant (có Hybrid Search) ban đầu lại cho kết quả **Completeness** tệ hơn bản Baseline đơn giản. Giả thuyết ban đầu của tôi là càng thêm nhiều kỹ thuật hiện đại thì kết quả phải càng tốt. Tuy nhiên thực tế cho thấy, việc kết hợp thêm BM25 mà không có Reranker đã làm "nhiễu" danh sách Top 3 kết quả, khiến AI nhận được những đoạn văn bản chứa nhiều từ khóa nhưng lại thiếu thông tin bao quát.


---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** "Approval Matrix để cấp quyền hệ thống là tài liệu nào?" (Câu q07 trong `test_questions.json`)

**Phân tích:** 
Đây là một câu hỏi thuộc mức độ "Hard" vì tên tài liệu mà người dùng hỏi ("Approval Matrix") đã cũ và được đổi tên thành "Access Control SOP" trong bộ dữ liệu hiện tại. 

*   **Kết quả Baseline:** Trả lời khá tốt (điểm 5/5/5/3). Nhờ vào Vector Search (Dense), hệ thống tìm được file `access-control-sop.md` dựa trên ngữ nghĩa tương đồng về việc "cấp quyền". Tuy nhiên điểm Completeness chỉ đạt 3 vì AI chưa nêu bật được mối liên hệ giữa tên cũ và tên mới.
*   **Lỗi nằm ở đâu?**: Lỗi nằm ở phần **Retrieval**. Bản Hybrid Search ban đầu (Variant) đã ưu tiên các đoạn văn bản có từ "Approval" và "Matrix" rời rạc, làm đẩy trôi đoạn văn bản có chứa câu: *"Tài liệu này trước đây được gọi là Approval Matrix"* xuống hạng thấp hơn (ngoài Top 3).
*   **Sự cải thiện của Variant (sau Rerank):** Sau khi tôi triển khai **Cross-Encoder Reranker**, kết quả đã được cải thiện rõ rệt. Reranker đã "đọc" kỹ 20 candidate đầu tiên và nhận diện ra đoạn chứa thông tin lịch sử đổi tên mới là quan trọng nhất, đẩy nó lên vị trí số 1. Nhờ vậy, điểm Completeness của câu này đã tăng lên đáng kể, chứng minh được sức mạnh của việc sắp xếp lại kết quả tìm kiếm trong RAG.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

Nếu có thêm thời gian, tôi sẽ thử nghiệm kỹ thuật **Query Transformation (HyDE - Hypothetical Document Embeddings)**. Do dữ liệu chính sách thường rất khô khan và ngắn gọn, trong khi câu hỏi của người dùng lại mang tính diễn giải, tôi muốn dùng LLM để tạo ra một "tài liệu giả định" từ câu hỏi trước khi mang đi tìm kiếm. Điểm số Completeness trong bài test vừa rồi cho thấy AI vẫn thiếu một chút ngữ cảnh mở rộng, và HyDE có thể là lời giải để tăng cường khả năng kết nối giữa câu hỏi tự nhiên và văn bản chính sách cứng nhắc.

---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*
*Ví dụ: `reports/individual/nguyen_van_a.md`*
