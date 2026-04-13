# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Đào Danh Đăng Phụng
**Vai trò trong nhóm:** Eval Owner  
**Ngày nộp:** 13/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

Trong lab này, tôi đảm nhận vai trò Eval Owner với mục tiêu xây dựng quy trình đánh giá khách quan cho pipeline RAG thay vì chỉ nhìn cảm tính vào một vài câu trả lời mẫu. Trước hết, tôi hoàn thiện bộ `grading_questions.json` gồm 10 câu khó, tập trung vào các tình huống thường làm hệ thống RAG sai như cross-document reasoning, temporal/version reasoning, exception chain và insufficient context (cần abstain). Song song đó, tôi rà soát bộ `test_questions.json` để đảm bảo dữ liệu test nền vẫn ổn định và có thể dùng so sánh A/B.

Tiếp theo, tôi phối hợp kiểm chứng thủ công các câu và đối chiếu trực tiếp với tài liệu trong `data/docs` để xác nhận `expected_answer` và `expected_sources` có grounding. Cuối cùng, tôi cập nhật luồng chạy `eval.py` để có thể chạy riêng theo dataset `test` và `grading`, xuất scorecard tách biệt cho baseline/variant, giúp cả nhóm đọc kết quả rõ ràng hơn theo từng mục tiêu đánh giá.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

Điều tôi hiểu rõ nhất sau bài này là: một bộ benchmark tốt không chỉ đo "trả lời đúng fact", mà còn phải đo được "khả năng chống sai". Trước đây tôi nghĩ chỉ cần các câu hỏi fact đơn giản là đủ để so sánh model/retriever, nhưng thực tế cho thấy pipeline có thể đạt điểm rất cao ở câu easy mà vẫn thất bại ở câu cần tổng hợp nhiều điều kiện hoặc cần từ chối trả lời khi thiếu dữ liệu.

Tôi cũng hiểu rõ hơn sự khác nhau giữa hai lớp đánh giá: `test_questions` để đo hiệu năng cơ bản và độ ổn định chung; `grading_questions` để stress-test các failure mode cụ thể. Hai bộ này không thay thế nhau mà bổ trợ nhau. Nếu chỉ giữ một bộ, nhóm dễ kết luận sai về chất lượng thật của hệ thống. Với vai trò Eval Owner, việc tách lớp benchmark như vậy giúp tôi giải thích kết quả A/B minh bạch hơn và chỉ ra điểm yếu đúng bản chất (retrieval, synthesis hay abstain behavior).

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

Khó khăn lớn nhất tôi gặp là hiện tượng điểm số "đẹp nhưng giả", cụ thể có lúc cả baseline và variant đều ra pattern gần như giống nhau trên toàn bộ câu hỏi. Khi đào log chi tiết, tôi thấy nguyên nhân không nằm ở thuật toán retrieval mà do lỗi tầng gọi LLM/API (401 key, 429 quota). Khi generator hoặc judge bị lỗi, nhiều metric rơi về fallback score, làm kết quả nhìn rất đều nhưng không phản ánh năng lực thật của pipeline.

Điều này khiến tôi ngạc nhiên vì nếu chỉ nhìn file scorecard cuối mà không đọc terminal log, rất dễ kết luận sai rằng "hai cấu hình ngang nhau". Từ trải nghiệm này, tôi rút ra nguyên tắc cho evaluation: luôn kiểm tra điều kiện vận hành (API key/quota, retry behavior, error handling) trước khi diễn giải metric. Nói cách khác, chất lượng đánh giá phụ thuộc mạnh vào độ tin cậy của hạ tầng chạy test, không chỉ phụ thuộc vào thiết kế câu hỏi.

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** "Khi làm việc remote, tôi phải dùng VPN và được kết nối trên tối đa bao nhiêu thiết bị?" (`gq02` trong `grading_questions.json`)

**Phân tích:**  
Tôi chọn `gq02` vì đây là dạng câu điển hình để đo năng lực **multi-document synthesis**. Để trả lời đúng hoàn chỉnh, hệ thống phải lấy được ít nhất hai mảnh thông tin từ hai nguồn khác nhau: (1) chính sách HR nêu VPN là bắt buộc khi remote; (2) FAQ IT nêu giới hạn 2 thiết bị và tên phần mềm Cisco AnyConnect. Nếu chỉ retrieve một nguồn thì câu trả lời dễ đúng một nửa nhưng thiếu completeness.

Ở kết quả baseline, context recall cho câu này vẫn cao (retrieved đúng source), nhưng phần answer thường chỉ nêu "2 thiết bị", thiếu phần "VPN bắt buộc" và đôi khi không nêu tên phần mềm. Điểm này cho thấy vấn đề không chỉ nằm ở retrieval mà còn ở cách generation tóm tắt thông tin: model ưu tiên trả lời ngắn theo ý người dùng thấy rõ nhất, dẫn đến thiếu các key points cần chấm. Với tư cách Eval Owner, tôi đánh dấu đây là failure mode quan trọng vì nó phản ánh chênh lệch giữa "đúng một phần" và "đúng đầy đủ có căn cứ", điều mà bộ `grading_questions` được thiết kế để bắt.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

Nếu có thêm thời gian, tôi sẽ mở rộng evaluation theo hướng "rubric-aware scoring": dùng trực tiếp `grading_criteria` và `points` trong `grading_questions.json` để chấm weighted score theo từng kỹ năng RAG (freshness, cross-doc, abstain). Đồng thời, tôi sẽ thêm bước fail-fast khi gặp lỗi API/quota để tránh scorecard bị nhiễu bởi fallback score. Mục tiêu là giúp kết quả A/B vừa công bằng, vừa đủ chi tiết để nhóm quyết định hướng tuning tiếp theo.

---


