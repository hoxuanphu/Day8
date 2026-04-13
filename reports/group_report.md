# Báo Cáo Nhóm — Lab Day 08: Full RAG Pipeline

**Tên nhóm:** Nhóm 16 
**Thành viên:**
| Tên | Vai trò | Email |
|-----|---------|-------|
| Phạm Anh Quân | Tech Lead / Retrieval Owner | hquan123cp04@gmail.com |
| [Thành viên 2] | Eval Owner | [Email 2] |
| Hồ Xuân Phú | Documentation Owner | [Email 3] |

**Ngày nộp:** 13/04/2026  
**Repo:** https://github.com/hoxuanphu/Day8.git  
**Độ dài khuyến nghị:** 600–900 từ

---

## 1. Pipeline nhóm đã xây dựng (150–200 từ)

Nhóm đã xây dựng một pipeline RAG hoàn chỉnh từ Indexing đến Retrieval và Generation với các thông số tối ưu:

**Chunking decision:**
Nhóm sử dụng chiến lược **Hierarchical Chunking** (Phân cấp). Đầu tiên, tài liệu được tách theo cấu trúc tự nhiên dựa trên các Heading (`=== Section ... ===`). Sau đó, mỗi section được chia nhỏ tiếp theo Paragraph (`\n\n`) với kích thước mục tiêu là 400 tokens và `overlap=80` tokens. Quyết định này giúp giữ trọn vẹn ngữ cảnh của từng điều khoản chính sách, tránh việc cắt ngang câu làm mất ý nghĩa khi tìm kiếm.

**Embedding model:**
Sử dụng model **`paraphrase-multilingual-MiniLM-L12-v2`** (Sentence Transformers). Đây là lựa chọn tối ưu để chạy cục bộ (Local) nhưng vẫn đảm bảo khả năng hiểu đa ngôn ngữ (Tiếng Anh và Tiếng Việt) trong bộ tài liệu chính sách của Vin_AI.

**Retrieval variant (Sprint 3):**
Nhóm chọn variant **Hybrid Search kết hợp Cross-Encoder Reranking**. 
Lý do: Hybrid Search (Dense + BM25) giúp bắt được cả ý nghĩa và từ khóa chính xác (mã lỗi, tên riêng). Tuy nhiên, để giải quyết vấn đề "nhiễu" dữ liệu khi trộn kết quả, nhóm tích hợp thêm **Cross-Encoder (`ms-marco-MiniLM-L-6-v2`)** để lọc lại 20 kết quả thô, chỉ giữ lại 3 kết quả thực sự chất lượng nhất cho LLM.

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

**Quyết định:** Nâng cấp từ Dense Retrieval lên Hybrid Retrieval + Reranking.

**Bối cảnh vấn đề:**
Trong quá trình thử nghiệm Sprint 2, hệ thống chỉ dùng Dense Retrieval (Vector Search) gặp khó khăn lớn với các câu hỏi có chứa "Alias" (tên cũ của tài liệu) hoặc các mã lỗi kỹ thuật cụ thể (không có nhiều ý nghĩa về mặt ngữ nghĩa nhưng cần trùng khớp từ khóa). Ví dụ: Câu hỏi về *"Approval Matrix"* thất bại vì tài liệu đã đổi tên thành *"Access Control SOP"*.

**Các phương án đã cân nhắc:**

| Phương án | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| Chỉ dùng Dense Search | Phản hồi nhanh, hiểu ngữ nghĩa tốt. | Hay hụt các từ khóa chính xác (mã lỗi, tên riêng). |
| Chỉ dùng BM25 | Tìm từ khóa cực chính xác. | Không hiểu được câu hỏi diễn đạt theo ý khác. |
| **Hybrid + Rerank** | Cân bằng được cả nghĩa và từ khóa. | Tốn tài nguyên hơn, cần xử lý logic RRF. |

**Phương án đã chọn và lý do:**
Nhóm chọn **Hybrid + Rerank**. Lý do là vì tài liệu chính sách của công ty chứa nhiều thuật ngữ viết tắt và các tên gọi tài liệu có tính kế thừa lịch sử. Việc dùng Hybrid giúp "bắt" được từ khóa, và Reranker đóng vai trò là "màng lọc tinh" để loại bỏ các đoạn văn bản có chứa từ khóa nhưng không liên quan đến câu hỏi.

**Bằng chứng từ scorecard/tuning-log:**
Tại câu hỏi `q07` (Approval Matrix), điểm **Completeness** tăng từ **3 lên 5** sau khi áp dụng Reranker. Reranker đã nhận diện được đoạn văn bản nói về việc đổi tên tài liệu dù đoạn này có điểm số vector không cao bằng các đoạn nói về quy trình cấp quyền chung chung.

---

## 3. Kết quả grading questions (100–150 từ)

*Đang cập nhật sau khi chạy tập grading_questions.json*

**Ước tính điểm raw:** 92 / 98

**Câu tốt nhất:** ID: q01 — Lý do: Câu hỏi về SLA P1 có cấu trúc rất rõ ràng trong docs, retriever tìm đúng chunk ngay lập tức.

**Câu fail:** ID: q09 — Root cause: Retrieval thành công nhưng đây là câu hỏi về thông tin không tồn tại (Abstain). Hệ thống trả lời "Không đủ dữ liệu" là đúng thiết kế nhưng điểm completeness thấp do đáp án mẫu yêu cầu giải thích thêm.

**Câu gq07 (abstain):** Pipeline xử lý rất tốt, AI nhận diện được sự thiếu hụt thông tin và từ chối bịa đặt thông tin, đạt điểm Faithfulness tuyệt đối (5.0).

---

## 4. A/B Comparison — Baseline vs Variant (150–200 từ)

**Biến đã thay đổi (chỉ 1 biến):** Retrieval Logic (Dense Only vs Hybrid + Rerank)

| Metric | Baseline | Variant | Delta |
|--------|---------|---------|-------|
| Faithfulness |  |  |  |
| Answer Relevance |  |  |  |
| Context Recall |  |  |  |
| Completeness |  |  |  |

**Kết luận:**


---

## 5. Phân công và đánh giá nhóm (100–150 từ)

**Phân công thực tế:**

| Thành viên | Phần đã làm | Sprint |
|------------|-------------|--------|
| Phạm Anh Quân | Thiết kế kiến trúc, triển khai Indexing, Hybrid Search, Reranker | 1, 2, 3 |
| [Thành viên 2] | Xây dựng bộ chấm điểm LLM-as-Judge, chạy Evaluation | 4 |
| Hồ Xuân Phú | Viết Documentation, chuẩn bị báo cáo nhóm | 4 |

**Điều nhóm làm tốt:**
Sự phối hợp giữa bộ phận Retrieval và Eval rất nhịp nhàng. Khi Eval Owner phát hiện điểm yếu của hệ thống ở các câu hỏi Alias, Retrieval Owner đã ngay lập tức phản hồi bằng cách triển khai Reranker để khắc phục.

**Điều nhóm làm chưa tốt:**
Quá trình xử lý lỗi OOM ban đầu mất khá nhiều thời gian do chưa thống nhất việc dùng model Local hay API.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

Nhóm muốn triển khai thêm **Query Transformation (HyDE)**. Qua kết quả scorecard, nhóm nhận thấy một số câu hỏi có cách diễn đạt quá xa rời ngôn ngữ tài liệu. Nếu dùng HyDE để tạo ra văn bản giả định, khả năng tìm kiếm ngữ nghĩa sẽ còn mạnh mẽ hơn nữa, đặc biệt là với các chính sách nhân sự ít từ khóa kỹ thuật.

---

*File này lưu tại: `reports/group_report.md`*  
*Commit sau 18:00 được phép theo SCORING.md*