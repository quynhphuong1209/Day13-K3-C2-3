# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm: Nhóm 3
- Repository URL: https://github.com/quynhphuong1209/Day13-K3-Observability
- Commit SHA cuối: (Sẽ điền sau khi commit)
- Thành viên và vai trò:
  - Nguyễn Văn A (Backend & Security Engineer): CP0 & CP1 (Logging, Correlation ID, Context Enrichment, PII Scrubbing)
  - Kim Mạnh Hùng (SRE, Tracing & Dashboard Engineer): CP2 (Langfuse Tracing, Prompt Versioning, Dashboard Spec, SLO & Alert Runbook)
  - Lê Văn C (QA & Chief Incident Investigator): CP0, CP3 (Load Test, Audit Git, Incident Investigation & REPORT.md)

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: 100/100
- Tổng số traces: 10 traces
- Số PII leak còn lại: 0
- Link/đường dẫn dashboard: config/dashboard.yaml (đường dẫn config) và submission/evidence/validate_dashboard.png (ảnh kiểm định)

## 3. Logging và tracing

- Evidence correlation ID: [correlation_id_log.json](file:///d:/Day13_2A202601679_KimManhHung/submission/evidence/correlation_id_log.json)
- Evidence PII redaction: [pii_redact_log.json](file:///d:/Day13_2A202601679_KimManhHung/submission/evidence/pii_redact_log.json)
- Evidence trace waterfall: [trace_waterfall.png](file:///d:/Day13_2A202601679_KimManhHung/submission/evidence/trace_waterfall.png)
- Giải thích một span đáng chú ý: Span `retrieve` (thuộc service RAG) và `generate` (thuộc service LLM) được trang bị decorator `@observe(as_type="span")` giúp hiển thị rõ ràng timeline trễ của từng bước. Trong điều kiện bình thường, `retrieve` tốn khoảng 0ms đến 10ms, còn `generate` (LLM sinh câu trả lời) chiếm phần lớn thời gian xử lý (~150ms).

## 4. Prompt versioning

- Prompt name: `day13-chat`
- Version/label baseline: v1 (gắn label `baseline` & `production`)
- Version/label candidate: v2 (gắn label `candidate`)
- Trace ID của mỗi version:
  - Trace ID v1: Ví dụ từ logs/traces
  - Trace ID v2: Ví dụ từ logs/traces
- Bằng chứng đổi label hoặc rollback: [prompt_v1_v2.png](file:///d:/Day13_2A202601679_KimManhHung/submission/evidence/prompt_v1_v2.png) và [prompt_rollback.png](file:///d:/Day13_2A202601679_KimManhHung/submission/evidence/prompt_rollback.png)

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`: HỢP LỆ: 6/6 panel
- Evidence dashboard: [validate_dashboard.png](file:///d:/Day13_2A202601679_KimManhHung/submission/evidence/validate_dashboard.png)
- SLO đã chọn và lý do:
  - Latency P95 ≤ 3000ms: Bảo đảm 95% request phản hồi trong 3 giây để tối ưu hóa trải nghiệm người dùng chat.
  - Error rate ≤ 2%: Hạn chế tối đa các lỗi hệ thống HTTP 500 ảnh hưởng tới người dùng.
  - Daily cost ≤ $2.5 USD: Kiểm soát chi phí vận hành gọi API LLM, tránh tình trạng spam vọt ngân sách.
  - Quality average ≥ 0.75: Đảm bảo độ chính xác và chất lượng nội dung câu trả lời.
- Alert rules và runbook: [config/alert_rules.yaml](file:///d:/Day13_2A202601679_KimManhHung/config/alert_rules.yaml) và [docs/alerts.md](file:///d:/Day13_2A202601679_KimManhHung/docs/alerts.md)

## 6. Điều tra challenge

*(Sẽ điền sau khi nhận challenge và thực hiện điều tra ở Checkpoint 3)*

- Challenge ID:
- Triệu chứng từ metrics:
- Trace ID liên quan:
- Log line/correlation ID liên quan:
- Root cause:
- Fix action:
- Preventive measure:

## 7. Đóng góp cá nhân

| Thành viên | Phần việc | Commit/PR | Điều đã học |
|---|---|---|---|
| Nguyễn Văn A | CP1: Correlation ID Middleware, Context Enrichment, PII Scrubbing đệ quy & Regex patterns mới. | `feat(logging): add correlation id middleware and pii scrubber` | Hiểu cơ chế share contextvars trong ứng dụng async, cấu trúc JSONL logging và tầm quan trọng của việc che PII trước khi lưu trữ. |
| Trần Thị B (Kim Mạnh Hùng) | CP2: Tích hợp Langfuse Traces metadata, Prompt Versioning & Rollback, bổ sung Error Rate Metrics, SLO & Alert Runbook. | `feat(observability): setup langfuse tracing, prompt versioning and alert rules` | Hiểu cách thiết kế Cảnh báo dựa trên triệu chứng (Symptom-based alert), quy trình quản lý vòng đời Prompt và ý nghĩa chỉ số P95/P99 latency. |
| Lê Văn C | CP0/CP2 Load testing, CP3 Chủ trì điều tra Incident Challenge theo luồng M->T->L, Audit Git & Hoàn thiện REPORT.md. | `docs(report): complete incident investigation and submission evidence` | Nắm vững quy trình khoanh vùng sự cố thực tế từ chỉ số tổng hợp (Metrics) đến hành trình request (Traces) và nguyên nhân gốc (Logs). |
