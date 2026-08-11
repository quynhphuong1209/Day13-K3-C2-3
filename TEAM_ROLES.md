# Kế hoạch Phân chia Vai trò & Nhiệm vụ Nhóm 3 Thành viên
## Lab 13 — Observability cho hệ thống AI: Metrics, Traces & Logs

---

## 📊 1. Cấu trúc Điểm số & Mục tiêu Nhóm 3 người

* **Tổng điểm (100 điểm)** = **60 điểm Nhóm** + **40 điểm Cá nhân** (+ Tối đa **10 điểm Bonus**).
* **Điểm Nhóm (60đ)**:
  * **A1 (30đ)**: Triển khai kỹ thuật (Logging, Correlation ID, PII, Traces, Prompt Versioning, Dashboard, SLO, Alert Rules).
  * **A2 (10đ)**: Điều tra Incident theo luồng `Metrics → Traces → Logs` (Xác định triệu chứng, root cause, fix action, preventive measure).
  * **A3 (20đ)**: Demo trực tiếp & Trả lời giải thích hệ thống.
* **Điểm Cá nhân (40đ)**:
  * **B1 (20đ)**: Mức độ hiểu bài & Trả lời phỏng vấn cá nhân về phần việc phụ trách.
  * **B2 (20đ)**: Bằng chứng đóng góp Git commit/PR khớp chính xác với khai báo trong `submission/REPORT.md`.

---

## 👥 2. Sơ đồ Ma trận Phân công Vai trò (Role Assignment Matrix)

| Vai trò & Chức danh | Mốc công việc (CP) | File code / config phụ trách | Evidence cần thu thập |
| :--- | :--- | :--- | :--- |
| **Thành viên A**<br>`Backend & Security Engineer` | **CP0 & CP1**<br>Logging, Correlation ID, Context Enrichment, PII Scrubbing | • `app/middleware.py`<br>• `app/main.py`<br>• `app/logging_config.py`<br>• `app/pii.py` | • `validate_logs.png` (≥ 80/100)<br>• `correlation_id_log.json`<br>• `pii_redact_log.json` |
| **Thành viên B**<br>`SRE, Tracing & Dashboard Engineer` | **CP2**<br>Langfuse Tracing, Prompt Versioning, Dashboard Spec, SLO & Alert Runbook | • `app/agent.py`<br>• `app/metrics.py`<br>• `config/slo.yaml`<br>• `config/alert_rules.yaml`<br>• `docs/alerts.md`<br>• `docs/dashboard-spec.md` | • `langfuse_traces.png` (≥ 10 traces)<br>• `trace_waterfall.png`<br>• `prompt_v1_v2.png`<br>• `prompt_rollback.png`<br>• `validate_dashboard.png`<br>• `dashboard.png` |
| **Thành viên C**<br>`QA & Chief Incident Investigator` | **CP0, CP3 & Nộp bài**<br>Load Test, Dẫn dắt Điều tra Challenge, Audit Git & Tổng hợp Báo cáo | • `scripts/load_test.py`<br>• `submission/REPORT.md`<br>• Thư mục `submission/evidence/` | • `challenge_metrics_symptom.png`<br>• `challenge_trace_span.png`<br>• `challenge_log_rootcause.json`<br>• File `submission/REPORT.md` hoàn chỉnh |

---

## 🛠 3. Chi tiết Nhiệm vụ từng Thành viên theo Checkpoint

### 🟢 Checkpoint 0: Setup & Baseline (0:00 – 0:30)
* **Cả 3 thành viên**:
  1. Fork repo `Day13-K3-Observability` về GitHub nhóm.
  2. Clone repo về máy local, khởi tạo virtual environment (`python -m venv .venv`).
  3. Cài đặt dependencies (`pip install -r requirements.txt`).
  4. Tạo `.env` từ `.env.example`, điền API credentials của Langfuse (`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`).
* **Thành viên A**: Khởi động API server (`uvicorn app.main:app --reload --env-file .env`).
* **Thành viên C**:
  1. Chạy load test sinh log ban đầu (`python scripts/load_test.py`).
  2. Chạy validator lấy điểm baseline (`python scripts/validate_logs.py`) và lưu kết quả vào báo cáo.

---

### 🔵 Checkpoint 1: Structured Logging, Correlation ID & PII (0:30 – 1:30)
👉 **Thành viên A dẫn dắt chính**:

1. **Middleware Correlation ID (`app/middleware.py`)**:
   * Xóa contextvars cũ bằng `clear_contextvars()` ở đầu `dispatch()` để tránh rò rỉ dữ liệu giữa các request.
   * Lấy header `x-request-id` hoặc sinh mới format `req-<8hex>` (`uuid.uuid4().hex[:8]`).
   * Bind `correlation_id` vào `structlog` contextvars và `request.state`.
   * Trả về header `x-request-id` và `x-response-time-ms` trong HTTP response.
   * *(Mở rộng)* Thêm Generic Exception Handler trong `app/main.py` để khi xảy ra lỗi 500 (`tool_fail`), HTTP response header vẫn chứa `x-request-id`.

2. **Context Enrichment (`app/main.py`)**:
   * Trong hàm `chat()`, gọi `bind_contextvars()` gán các metadata: `user_id_hash` (dùng `hash_user_id()`), `session_id`, `feature`, `model`, `env`.

3. **PII Scrubbing (`app/logging_config.py` & `app/pii.py`)**:
   * In-uncomment processor `scrub_event` trong `logging_config.py` (vị trí: sau `TimeStamper` và trước `JSONRenderer`).
   * Bổ sung regex patterns mới trong `app/pii.py`: `passport` (Hộ chiếu) và `address_vn` (Địa chỉ Việt Nam).
   * Nâng cấp hàm `scrub_event` để duyệt đệ quy qua toàn bộ chuỗi string và dictionary trong event log.

4. **Nghiệm thu & Evidence**:
   * Xóa log cũ, chạy lại `load_test.py`, chạy `python scripts/validate_logs.py` đạt điểm **≥ 80/100**.
   * Chụp ảnh màn hình điểm validator và trích xuất log dòng mẫu có `correlation_id` + `[REDACTED_...]` lưu vào `submission/evidence/`.

---

### 🟡 Checkpoint 2: Metrics, Traces, Dashboard & Alerts (1:30 – 2:30)
👉 **Thành viên B dẫn dắt chính**:

1. **Langfuse Trace Metadata & Correlation ID (`app/agent.py`)**:
   * Import `get_contextvars` từ `structlog.contextvars`.
   * Cập nhật trace Langfuse đính kèm `correlation_id` vào `metadata` (`metadata={"correlation_id": ...}`).
   * *(Mở rộng)* Gắn decorator `@observe(as_type="span")` cho `retrieve()` trong `app/mock_rag.py` và `generate()` trong `app/mock_llm.py` để hiển thị waterfall chi tiết RAG / LLM.

2. **Prompt Versioning (`docs/PROMPT_VERSIONING.md`)**:
   * Tạo prompt `day13-chat` trên Langfuse với 3 biến: `Feature={{feature}}`, `Docs={{docs}}`, `Question={{message}}`.
   * Đặt v1 (gắn label `baseline` & `production`) và v2 (gắn label `candidate`).
   * Đổi label `production` sang v2 và thực hiện rollback về v1. Chụp lại ảnh chứng minh 2 trace ID và màn hình đổi label.

3. **Metrics & Dashboard (`app/metrics.py` & `docs/dashboard-spec.md`)**:
   * Thêm chỉ số `error_rate_pct` vào hàm `snapshot()` trong `app/metrics.py`.
   * Thiết kế Dashboard 6 nhóm chỉ số trong `docs/dashboard-spec.md` (Latency, Traffic, Error, Cost, Tokens, Quality).
   * Chạy `python scripts/validate_dashboard.py` báo thành công.

4. **SLO & Alert Rules (`config/slo.yaml`, `config/alert_rules.yaml`, `docs/alerts.md`)**:
   * Điều chỉnh `config/slo.yaml` và điền 3 alert rules dựa trên triệu chứng (Symptom-based) trong `config/alert_rules.yaml` (`high_latency_p95`, `elevated_error_rate`, `cost_budget_exceeded`).
   * Hoàn thiện tài liệu **Alert Runbook** trong `docs/alerts.md` đầy đủ 5 mục (Trigger condition, User impact, 3 bước kiểm tra đầu tiên, Mitigation tạm thời, Owner).

---

### 🔴 Checkpoint 3: Challenge Incident Investigation (2:30 – 3:30)
👉 **Thành viên C dẫn dắt chính cùng cả nhóm**:

1. Nhận file `config/challenge.json` từ Lab Coach và chạy:
   ```bash
   python scripts/inject_incident.py
   python scripts/load_test.py --challenge --concurrency 5
   ```

2. Thực hiện quy trình điều tra 3 bước `Metrics → Traces → Logs`:
   * **Bước 1 (Metrics)**: Gọi `curl http://localhost:8000/metrics | python -m json.tool` phát hiện chỉ số bất thường (latency p95 tăng, error rate cao hoặc cost spike).
   * **Bước 2 (Traces)**: Mở Langfuse, khoanh vùng trace bị chậm/lỗi, đọc `correlation_id` từ metadata và kiểm tra waterfall span.
   * **Bước 3 (Logs)**: Dùng `correlation_id` vừa tìm được lọc trong `data/logs.jsonl` bằng Python/jq để tìm nguyên nhân gốc (log `request_failed` hoặc exception).

3. Tổng hợp kết quả điền vào Mục 6 của `submission/REPORT.md`:
   * Challenge ID
   * Triệu chứng từ Metrics
   * Trace ID liên quan
   * Log line / Correlation ID liên quan
   * Root Cause
   * Fix Action & Preventive Measure

---

### 🏁 Checkpoint 4: Báo cáo & Nộp bài (3:30 – 4:00)
👉 **Thành viên C chủ trì tổng hợp, Thành viên A & B kiểm tra chéo**:

* **Thành viên C**:
  * Điền đầy đủ thông tin nhóm, kết quả kỹ thuật, link bằng chứng, SLO/Alerts và đóng góp cá nhân trong `submission/REPORT.md`.
  * Chạy các lệnh rà soát cuối cùng:
    ```bash
    python -m pytest -q
    python scripts/validate_logs.py
    python scripts/validate_dashboard.py
    git status --short
    ```
* **Cả 3 thành viên**: Commit code và tài liệu cá nhân lên Git với message rõ ràng, đảm bảo commit author khớp với tên khai báo trong report.

---

## 📋 4. Khai báo Bảng Đóng góp Cá nhân cho `submission/REPORT.md`

Dán bảng bên dưới vào **Mục 7 (Đóng góp cá nhân)** trong file `submission/REPORT.md`:

```markdown
| Thành viên | Phần việc | Commit/PR | Điều đã học |
|---|---|---|---|
| Nguyễn Văn A | CP1: Correlation ID Middleware (`clear_contextvars`), Context Enrichment, PII Scrubbing processor đệ quy & Regex patterns mới. | `feat(logging): add correlation id middleware and pii scrubber` | Hiểu cơ chế share contextvars trong ứng dụng async, cấu trúc JSONL logging và tầm quan trọng của việc che PII trước khi lưu trữ. |
| Trần Thị B | CP2: Tích hợp Langfuse Traces metadata, Prompt Versioning & Rollback, bổ sung Error Rate Metrics, SLO & Alert Runbook. | `feat(observability): setup langfuse tracing, prompt versioning and alert rules` | Hiểu cách thiết kế Cảnh báo dựa trên triệu chứng (Symptom-based alert), quy trình quản lý vòng đời Prompt và ý nghĩa chỉ số P95/P99 latency. |
| Lê Văn C | CP0/CP2 Load testing, CP3 Chủ trì điều tra Incident Challenge theo luồng M->T->L, Audit Git & Hoàn thiện REPORT.md. | `docs(report): complete incident investigation and submission evidence` | Nắm vững quy trình khoanh vùng sự cố thực tế từ chỉ số tổng hợp (Metrics) đến hành trình request (Traces) và nguyên nhân gốc (Logs). |
```

---

## 🎤 5. Bộ Câu hỏi Bảo vệ & Phản biện (Cho điểm A3 & B1 - Tối đa 40 điểm)

| Thành viên | Câu hỏi phản biện có thể gặp | Gợi ý trả lời chuẩn xác |
| :--- | :--- | :--- |
| **Thành viên A** | Tại sao bước gọi `clear_contextvars()` ở đầu Middleware lại bắt buộc? | Trong ứng dụng FastAPI/Uvicorn async, một luồng (thread/task) có thể được dùng lại để xử lý nhiều request khác nhau. Nếu không clear contextvars ở đầu request mới, contextvars của request cũ sẽ bị dính sang request mới, gây ra hiện tượng rò rỉ dữ liệu (data leakage) giữa các user. |
| **Thành viên B** | Tại sao Alert rules phải thiết kế dạng **Symptom-based** thay vì **Cause-based**? | Nguyên nhân lỗi bên trong (như lỗi hàm RAG hay ngắt kết nối database) có thể thay đổi liên tục khi ứng dụng nâng cấp. Cảnh báo theo triệu chứng (User-facing symptom) giúp thông báo ngay lập tức khi người dùng chịu ảnh hưởng (Latency P95 > 3s, Error Rate > 5%), đảm bảo không bỏ sót sự cố thực tế và tránh nhiễu cảnh báo. |
| **Thành viên C** | Hãy giải thích luồng điều tra Incident nhóm vừa thực hiện trong CP3? | Đi từ **Metrics** để phát hiện triệu chứng bất thường (ví dụ: latency P95 vọt lên 2.8s) → Mở **Langfuse Trace** lọc các request bị chậm, phát hiện span `retrieve` thuộc feature `search` kéo dài 2.5s → Lấy `correlation_id` của trace đó tra cứu trong **Logs** (`logs.jsonl`) để tìm chính xác sự kiện `rag_timeout_warning` chỉ ra root cause. |
