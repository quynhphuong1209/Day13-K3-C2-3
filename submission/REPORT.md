# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm: Nhóm C2-3 (K3)
- Repository URL: https://github.com/quynhphuong1209/Day13-K3-C2-3
- Commit SHA cuối: (Cập nhật SHA commit cuối khi push)
- Thành viên và vai trò:
  - Đoàn Minh Hiếu: Backend & Security Engineer (Logging, Correlation ID, PII Redaction)
  - Kim Mạnh Hưng (MSHV: 2A202601679): SRE, Tracing & Dashboard Engineer (Langfuse Tracing, Prompt Versioning, Dashboard Spec, SLO & Alerts)
  - Đinh Lê Quỳnh Phương (MSHV: 2A202601865): QA & Chief Incident Investigator (Load Testing, Incident Investigation Lead, Git Audit & Report Synthesis)

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: 30/100 (Baseline CP0) ➔ 100/100 (Sau CP1)
- Tổng số traces: ≥ 10 traces trên Langfuse
- Số PII leak còn lại: 0 (Đã redact Email, Phone, CCCD, Credit Card, Passport, Address)
- Link/đường dẫn dashboard: docs/dashboard-spec.md (xem bằng chứng tại submission/evidence/dashboard.png)

## 3. Logging và tracing

- Evidence correlation ID: submission/evidence/correlation_id_log.json
- Evidence PII redaction: submission/evidence/pii_redact_log.json
- Evidence trace waterfall: submission/evidence/trace_waterfall.png
- Giải thích một span đáng chú ý: Span `run` của agent đóng vai trò parent span chứa metadata `correlation_id`, `user_id_hash`, `session_id`. Các sub-span `retrieve` (truy xuất RAG) và `generate` (sinh văn bản LLM) phản ánh thời gian thực thi của từng thành phần con, giúp nhanh chóng khoanh vùng bước bị chậm khi có sự cố.

## 4. Prompt versioning

- Prompt name: day13-chat
- Version/label baseline: Version 1 (label: `baseline`, `production`)
- Version/label candidate: Version 2 (label: `candidate`)
- Trace ID của mỗi version:
  - Version 1 trace ID: `trace-prompt-v1-baseline`
  - Version 2 trace ID: `trace-prompt-v2-candidate`
- Bằng chứng đổi label hoặc rollback: submission/evidence/prompt_rollback.png

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`: Hợp lệ (100% khớp contract config/dashboard.yaml)
- Evidence dashboard: submission/evidence/dashboard.png & submission/evidence/validate_dashboard.png
- SLO đã chọn và lý do: 
  - `latency_p95_ms` < 3000ms (99.5% requests): Đảm bảo trải nghiệm phản hồi nhanh cho người dùng.
  - `error_rate_pct` < 2% (99.0% requests): Duy trì độ tin cậy của AI API service.
  - `daily_cost_usd` < $2.5 (100%): Kiểm soát ngân sách vận hành mô hình.
- Alert rules và runbook: Đã cấu hình 3 Symptom-based Alerts trong `config/alert_rules.yaml` kèm Runbook chi tiết trong `docs/alerts.md`.

## 6. Điều tra challenge

- Challenge ID: day13-k3-observability-v1
- Triệu chứng từ metrics: Đọc từ `/metrics`: `latency_p95` tăng đột biến từ mức baseline ~200-300ms lên 2750ms (vượt ngưỡng cam kết SLO 2000ms), trong khi `error_rate_pct` vẫn bằng 0.0% (xem submission/evidence/challenge_metrics_symptom.png).
- Trace ID liên quan: `trace-challenge-rag-slow-01` (xem submission/evidence/challenge_trace_span.png). Trace waterfall chỉ rõ parent span `run` bị kéo dài 2.75s, trong đó sub-span `retrieve` chiếm tới 2.5s.
- Log line/correlation ID liên quan: Correlation ID `req-c1a2b3d4` trong `data/logs.jsonl` (trích xuất tại submission/evidence/challenge_log_rootcause.json). Dòng log `response_sent` ghi nhận `latency_ms: 2752`, `feature: "refund"`.
- Root cause: Lỗi trễ phát sinh tại hàm truy xuất tài liệu `retrieve()` trong `app/mock_rag.py`. Khi cờ incident `rag_slow` được kích hoạt cho feature `refund`, hàm bị ngắt quãng bởi delay nhân tạo `time.sleep(2.5)` khiến toàn bộ request thuộc feature `refund` bị chậm 2.5 giây.
- Fix action: Tối ưu chỉ mục truy vấn của vector store, thiết lập timeout tối đa cho bước retrieval (ví dụ 500ms) kèm cơ chế circuit breaker fallback về bộ nhớ cache hoặc trả câu trả lời chung.
- Preventive measure: Thiết lập cảnh báo trễ theo từng span con (`retrieve` span > 500ms), triển khai APM monitoring cho vector database response time và bổ sung bài kiểm thử hiệu năng tự động (performance regression test) trong quy trình CI/CD.

## 7. Đóng góp cá nhân

Với mỗi thành viên, ghi rõ nhiệm vụ và link commit/PR tương ứng.

| Thành viên | Phần việc | Commit/PR | Điều đã học |
|---|---|---|---|
| Đoàn Minh Hiếu | CP1: Correlation ID Middleware (`clear_contextvars`), Context Enrichment, PII Scrubbing processor đệ quy & Regex patterns mới. | `feat(logging): add correlation id middleware and pii scrubber` | Hiểu cơ chế share contextvars trong ứng dụng async, cấu trúc JSONL logging và tầm quan trọng của việc che PII trước khi lưu trữ. |
| Kim Mạnh Hưng (2A202601679) | CP2: Tích hợp Langfuse Traces metadata, Prompt Versioning & Rollback, bổ sung Error Rate Metrics, SLO & Alert Runbook. | `feat(observability): setup langfuse tracing, prompt versioning and alert rules` | Hiểu cách thiết kế Cảnh báo dựa trên triệu chứng (Symptom-based alert), quy trình quản lý vòng đời Prompt và ý nghĩa chỉ số P95/P99 latency. |
| Đinh Lê Quỳnh Phương (2A202601865) | CP0/CP2 Load testing, CP3 Chủ trì điều tra Incident Challenge theo luồng M->T->L, Audit Git & Hoàn thiện REPORT.md. | `docs(report): complete incident investigation and submission evidence` | Nắm vững quy trình khoanh vùng sự cố thực tế từ chỉ số tổng hợp (Metrics) đến hành trình request (Traces) và nguyên nhân gốc (Logs). |
