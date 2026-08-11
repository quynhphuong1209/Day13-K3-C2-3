# Runbook Cảnh báo Hệ thống (Alert & Runbook)

Tất cả các cảnh báo được thiết kế theo triệu chứng ảnh hưởng đến người dùng (Symptom-based) hoặc theo chỉ số SLO đã thỏa thuận.

---

## <a name="high_latency_p95"></a>1. high_latency_p95

- **Tên**: `high_latency_p95`
- **Severity**: `warning`
- **SLI/SLO liên quan**: Latency P95 (SLO: Latency P95 ≤ 3000ms)
- **Điều kiện và thời gian duy trì**: P95 Latency của endpoint `/chat` vượt quá 3000ms trong khoảng thời gian liên tục 5 phút.
- **Ảnh hưởng tới người dùng**: Người dùng gặp hiện tượng phản hồi chậm chạp khi chat với AI, giảm trải nghiệm sử dụng.
- **Ba bước kiểm tra đầu tiên**:
  1. Kiểm tra Langfuse Dashboard xem độ trễ bị tắc nghẽn ở đâu (Span RAG `retrieve` hay Span LLM `generate`).
  2. Xem Logs trong `data/logs.jsonl` tìm các log cảnh báo liên quan đến timeout (`rag_slow` hay `rag_timeout_warning`).
  3. Kiểm tra tình trạng tài nguyên hệ thống (CPU, RAM) và kết nối mạng bên ngoài đến API providers.
- **Mitigation tạm thời**:
  - Nếu do RAG slow, tạm thời tắt hoặc giảm số lượng tài liệu thu hồi hoặc chuyển sang cache.
  - Nếu do LLM, hạ cấp xuống model LLM nhẹ hơn/nhanh hơn hoặc bật chế độ dự phòng cục bộ (local fallback).
- **Owner**: SRE_Team

---

## <a name="elevated_error_rate"></a>2. elevated_error_rate

- **Tên**: `elevated_error_rate`
- **Severity**: `critical`
- **SLI/SLO liên quan**: Error Rate (SLO: Error Rate ≤ 2%)
- **Điều kiện và thời gian duy trì**: Tỷ lệ lỗi (Error Rate) của toàn bộ request vượt quá 2% trong cửa sổ giám sát 5 phút.
- **Ảnh hưởng tới người dùng**: Người dùng gặp lỗi hệ thống (HTTP 500), không thể thực hiện các cuộc trò chuyện với AI.
- **Ba bước kiểm tra đầu tiên**:
  1. Lọc log lỗi theo `event == "request_failed"` trong `data/logs.jsonl` và phân tích `error_type` (như `RuntimeError`, `Vector store timeout`, `LLM failure`).
  2. Lấy `correlation_id` của các trace bị lỗi trên Langfuse để xem vết waterfall cụ thể nhằm xác định chính xác cấu phần bị crash.
  3. Xác định xem có sự cố mất kết nối mạng diện rộng hoặc lỗi xác thực API Key của LLM/Langfuse hay không.
- **Mitigation tạm thời**:
  - Rollback code hoặc cấu hình về phiên bản ổn định gần nhất nếu sự cố xảy ra ngay sau khi deploy.
  - Chuyển hướng các request lỗi sang gateway/mô hình LLM dự phòng để duy trì tính sẵn sàng.
- **Owner**: SRE_Team

---

## <a name="cost_budget_exceeded"></a>3. cost_budget_exceeded

- **Tên**: `cost_budget_exceeded`
- **Severity**: `warning`
- **SLI/SLO liên quan**: Daily Cost Budget (SLO: Daily Cost ≤ $2.5 USD)
- **Điều kiện và thời gian duy trì**: Tổng chi phí tiêu thụ LLM tích lũy vượt quá $2.5 USD trong ngày.
- **Ảnh hưởng tới người dùng**: Không trực tiếp ảnh hưởng đến người dùng cuối ngay lập tức, nhưng đe dọa ngân sách vận hành của doanh nghiệp và có nguy cơ bị khóa tài khoản API do hết hạn mức.
- **Ba bước kiểm tra đầu tiên**:
  1. Kiểm tra panel `Cost over time` và `Input and output tokens` trên dashboard để xác định request nào tiêu tốn nhiều token nhất.
  2. Xem vết Langfuse Trace tìm các trace có `usage_details` bất thường (ví dụ: số token Completion vọt lên đột biến).
  3. Phân tích xem có hiện tượng loop/spam request hoặc lỗi logic dẫn đến gửi prompt cực lớn hay không (`cost_spike`).
- **Mitigation tạm thời**:
  - Áp dụng kỹ thuật Rate Limiting chặt chẽ hơn đối với các API endpoint để hạn chế spam.
  - Tạm thời chuyển sang mô hình LLM rẻ hơn (ví dụ: claude-haiku thay vì claude-sonnet) hoặc giới hạn số lượng token đầu ra (`max_tokens`).
- **Owner**: FinOps_Team
