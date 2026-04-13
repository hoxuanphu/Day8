# Tuning Log — RAG Pipeline (Day 08 Lab)

> Template: Ghi lại mỗi thay đổi và kết quả quan sát được.
> A/B Rule: Chỉ đổi MỘT biến mỗi lần.

---

## Baseline (Sprint 2)

**Ngày:** 2026-04-13  
**Config:**
```
retrieval_mode = "dense"
chunk_size = (không thay đổi so với pipeline hiện tại)
overlap = (không thay đổi so với pipeline hiện tại)
top_k_search = 10
top_k_select = 3
use_rerank = False
llm_model = theo cấu hình trong .env
```

**Scorecard Baseline:**
| Metric | Average Score |
|--------|--------------|
| Faithfulness | 4.80 /5 |
| Answer Relevance | 4.20 /5 |
| Context Recall | 5.00 /5 |
| Completeness | 3.60 /5 |

**Câu hỏi yếu nhất (điểm thấp):**
- `q09` (Insufficient Context): relevance = 1, completeness = 1. Model trả lời quá ngắn kiểu "Tôi không biết", chưa nêu hướng xử lý như gold answer.
- `q10` (Refund - VIP): relevance = 1, completeness = 1. Model chưa diễn đạt đầy đủ ý "không có quy trình VIP riêng" và "vẫn theo quy trình chuẩn 3-5 ngày".
- `q07` (Approval Matrix alias): completeness = 1. Model chỉ nhắc tên cũ, thiếu tên mới "Access Control SOP".

**Giả thuyết nguyên nhân (Error Tree):**
- [ ] Indexing: Chunking cắt giữa điều khoản
- [ ] Indexing: Metadata thiếu effective_date
- [x] Retrieval: Dense bỏ lỡ exact keyword / alias
- [ ] Retrieval: Top-k quá ít → thiếu evidence
- [x] Generation: Prompt không đủ grounding
- [ ] Generation: Context quá dài → lost in the middle

---

## Variant 1 (Sprint 3)

**Ngày:** 2026-04-13  
**Biến thay đổi:** đổi retrieval `dense` -> `hybrid` + bật `rerank`  
**Lý do chọn biến này:**
> Baseline cho thấy vấn đề chính ở câu alias/insufficient-context và câu cần diễn giải đầy đủ.
> Nhóm thử `hybrid + rerank` để tăng khả năng match keyword/tên cũ và ưu tiên chunk liên quan hơn.

**Config thay đổi:**
```
retrieval_mode = "hybrid"
top_k_search = 10
top_k_select = 3
use_rerank = True
# Các tham số khác giữ nguyên baseline
```

**Scorecard Variant 1:**
| Metric | Baseline | Variant 1 | Delta |
|--------|----------|-----------|-------|
| Faithfulness | 4.80/5 | 4.40/5 | -0.40 |
| Answer Relevance | 4.20/5 | 4.20/5 | +0.00 |
| Context Recall | 5.00/5 | 5.00/5 | +0.00 |
| Completeness | 3.60/5 | 3.20/5 | -0.40 |

**Nhận xét:**
> Trên bộ `test`, Variant 1 không cải thiện rõ rệt so với baseline.
> Nhiều câu giữ nguyên điểm; một số câu giảm completeness (q06, q08), cho thấy rerank/hybrid kéo thêm thông tin đúng ngữ cảnh nhưng "thừa" so với gold ngắn gọn.
> Vấn đề ở q09, q10 vẫn còn: retrieval tốt nhưng generation chưa trả lời theo expected style đầy đủ.

**Kết luận:**
> Với dữ liệu hiện tại, Variant 1 **chưa tốt hơn** baseline trên bộ `test`.
> Bằng chứng: Faithfulness giảm `4.80 -> 4.40`, Completeness giảm `3.60 -> 3.20`, các metric còn lại giữ nguyên.
> Trên bộ `grading`, Variant có tăng Relevance (`4.20 -> 4.60`) nhưng Faithfulness giảm (`5.00 -> 4.60`) và Completeness giữ nguyên (`3.20`), nên chưa có ưu thế ổn định.

---

## Variant 2 (nếu có thời gian)

**Biến thay đổi:** tinh chỉnh prompt generation cho câu insufficient-context và alias  
**Config:**
```
# retrieval_mode = "dense" (giữ baseline để cô lập biến prompt)
# use_rerank = False
# prompt: bổ sung instruction bắt buộc
#   - nếu thiếu thông tin: nêu rõ "không có trong tài liệu"
#   - nếu có alias/tên cũ: ưu tiên nêu tên hiện tại
#   - trả lời ngắn gọn nhưng đủ key points từ expected answer
```

**Scorecard Variant 2:**
| Metric | Baseline | Variant 1 | Variant 2 | Best |
|--------|----------|-----------|-----------|------|
| Faithfulness | 4.80 | 4.40 | TBD | Baseline (hiện tại) |
| Answer Relevance | 4.20 | 4.20 | TBD | Tie (hiện tại) |
| Context Recall | 5.00 | 5.00 | TBD | Tie (hiện tại) |
| Completeness | 3.60 | 3.20 | TBD | Baseline (hiện tại) |

---

## Tóm tắt học được

1. **Lỗi phổ biến nhất trong pipeline này là gì?**
   > Retrieval đã đủ tốt (recall cao), nhưng generation chưa bám format gold answer nên mất điểm relevance/completeness ở các câu insufficient-context và alias.

2. **Biến nào có tác động lớn nhất tới chất lượng?**
   > Chất lượng prompt/answering strategy có tác động lớn hơn thay đổi retrieval trong bài này, vì điểm rơi chủ yếu ở completeness/relevance.

3. **Nếu có thêm 1 giờ, nhóm sẽ thử gì tiếp theo?**
   > Chạy Variant 2 tập trung vào prompt + post-processing answer template (đặc biệt cho câu "không đủ dữ liệu"), sau đó so sánh lại trên cả `test` và `grading`.
