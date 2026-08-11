# ✅ CHECKLIST HOÀN THÀNH - LAB 13 OBSERVABILITY
## Nhóm C2-3 (K3) - Sẵn sàng Demo & Đạt 100/100

---

## 📊 TỔNG QUAN ĐIỂM SỐ

| Hạng mục | Điểm tối đa | Điểm đạt được | Trạng thái |
|---|---|---|---|
| **A1** - Triển khai kỹ thuật | 30 | 30/30 ✅ | Hoàn thành |
| **A2** - Điều tra Incident | 10 | 10/10 ✅ | Hoàn thành |
| **A3** - Demo & Giải thích | 20 | 18-20 🎯 | Sẵn sàng |
| **B1** - Hiểu bài cá nhân | 20 | 18-20 🎯 | Sẵn sàng |
| **B2** - Bằng chứng Git | 20 | 20/20 ✅ | Hoàn thành |
| **TỔNG CỘNG** | **100** | **96-100** | **🏆 Xuất sắc** |

---

## ✅ CHECKPOINT 0: SETUP & BASELINE

### Hoàn thành:
- [x] Fork repo về GitHub nhóm: https://github.com/quynhphuong1209/Day13-K3-C2-3
- [x] Clone repo, cài đặt dependencies
- [x] Tạo `.env` với Langfuse credentials
- [x] Khởi động API server thành công
- [x] Chạy baseline load test
- [x] Điểm baseline: 30/100 (ghi nhận)

### Evidence:
- Repository URL đã có trong REPORT.md
- Baseline score đã ghi nhận

---

## ✅ CHECKPOINT 1: LOGGING, CORRELATION ID & PII

### Thành viên phụ trách: Đoàn Minh Hiếu

### Hoàn thành:
- [x] **Middleware Correlation ID** (`app/middleware.py`)
  - [x] `clear_contextvars()` ở đầu request
  - [x] Generate/lấy `req-<8hex>` từ header
  - [x] Bind vào contextvars và request.state
  - [x] Trả về `x-request-id` và `x-response-time-ms` trong response header

- [x] **Context Enrichment** (`app/main.py`)
  - [x] `bind_contextvars()` với user_id_hash, session_id, feature, model, env
  - [x] `hash_user_id()` dùng SHA-256

- [x] **Generic Exception Handler** (`app/main.py`)
  - [x] Preserve correlation_id trong response 500
  - [x] Return `x-request-id` header ngay cả khi exception

- [x] **PII Scrubbing** (`app/logging_config.py` & `app/pii.py`)
  - [x] Uncomment processor `scrub_event`
  - [x] Bổ sung regex: `passport`, `address_vn`
  - [x] Scrubbing đệ quy toàn bộ event dict

### Kết quả:
- [x] `validate_logs.py`: **100/100** (tăng từ 30/100)
- [x] Zero PII leaks trong logs
- [x] Tất cả logs có correlation_id format `req-XXXXXXXX`
- [x] Enrichment đầy đủ: user_id_hash, session_id, feature, model, env

### Evidence:
- [x] `submission/evidence/correlation_id_log.json`
- [x] `submission/evidence/pii_redact_log.json`
- [x] `submission/evidence/validate_logs.png`

### Git Commit:
```
feat(logging): add correlation id middleware and pii scrubber
Author: Đoàn Minh Hiếu
```

---

## ✅ CHECKPOINT 2: METRICS, TRACES, DASHBOARD & ALERTS

### Thành viên phụ trách: Kim Mạnh Hưng (2A202601679)

### Hoàn thành:
- [x] **Langfuse Trace Metadata** (`app/agent.py`)
  - [x] Import `get_contextvars` từ structlog
  - [x] Bind `correlation_id` vào trace metadata
  - [x] Decorator `@observe(as_type="span")` cho `retrieve()` và `generate()`

- [x] **Prompt Versioning** (`docs/PROMPT_VERSIONING.md`)
  - [x] Tạo prompt `day13-chat` với 3 biến
  - [x] Version 1: label `baseline`, `production`
  - [x] Version 2: label `candidate`
  - [x] Rollback từ v2 về v1
  - [x] 2 trace IDs khác nhau cho v1 và v2

- [x] **Error Rate Metrics** (`app/metrics.py`)
  - [x] Thêm `error_rate_pct` vào hàm `snapshot()`
  - [x] Formula: `(total_errors / total_requests) * 100`

- [x] **Dashboard Specification** (`docs/dashboard-spec.md`)
  - [x] 6 panels: Latency, Traffic, Error, Cost, Tokens, Quality
  - [x] Mỗi panel có: data source, metrics, thresholds, SLO lines
  - [x] `validate_dashboard.py`: **6/6 panels hợp lệ**

- [x] **SLO Definition** (`config/slo.yaml`)
  - [x] 4 SLIs: latency_p95_ms (<3000), error_rate_pct (<2%), daily_cost_usd (<$2.5), quality_score_avg (>0.75)
  - [x] Percentile targets: 99.5%, 99.0%, 100%, 95.0%

- [x] **Alert Rules** (`config/alert_rules.yaml`)
  - [x] 3 Symptom-based alerts:
    - [x] `high_latency_p95`: P95 > 3000ms
    - [x] `elevated_error_rate`: error_rate > 2%
    - [x] `cost_budget_exceeded`: daily_cost > $2.5

- [x] **Alert Runbook** (`docs/alerts.md`)
  - [x] 3 runbooks đầy đủ 5 phần:
    - [x] Severity & Trigger condition
    - [x] User impact
    - [x] 3 bước kiểm tra đầu tiên
    - [x] Mitigation tạm thời
    - [x] Owner

### Kết quả:
- [x] Traces trên Langfuse: **≥10 traces**
- [x] Correlation ID xuất hiện trong trace metadata
- [x] Dashboard validated: **6/6 panels pass**
- [x] SLO & Alert rules đầy đủ theo lab requirements

### Evidence:
- [x] `submission/evidence/langfuse_traces.png`
- [x] `submission/evidence/trace_waterfall.png`
- [x] `submission/evidence/prompt_v1_v2.png`
- [x] `submission/evidence/prompt_rollback.png`
- [x] `submission/evidence/dashboard.png`
- [x] `submission/evidence/validate_dashboard.png`

### Git Commit:
```
feat(observability): setup langfuse tracing, prompt versioning and alert rules
Author: Kim Mạnh Hưng
```

---

## ✅ CHECKPOINT 3: INCIDENT INVESTIGATION

### Thành viên phụ trách: Đinh Lê Quỳnh Phương (2A202601865)

### Hoàn thành:
- [x] **Load Testing** (`scripts/load_test.py`)
  - [x] Chạy baseline load test
  - [x] Chạy challenge load test với `--challenge --concurrency 5`

- [x] **Challenge Investigation** (M→T→L flow)
  - [x] **Metrics**: Phát hiện latency P95 = 2750ms (vượt SLO 2000ms)
  - [x] **Traces**: Khoanh vùng span `retrieve` kéo dài 2.5s, feature="refund"
  - [x] **Logs**: Tìm correlation_id `req-c1a2b3d4`, confirm root cause

- [x] **Root Cause Analysis**
  - [x] Challenge ID: `day13-k3-observability-v1`
  - [x] Incident type: `rag_slow`
  - [x] Affected feature: `refund`
  - [x] Root cause: `retrieve()` trong `app/mock_rag.py` bị delay 2.5s khi incident active

- [x] **Fix Actions & Preventive Measures**
  - [x] Fix: Optimize vector store, timeout 500ms, circuit breaker fallback
  - [x] Preventive: Span-level alerts, APM monitoring, performance regression tests

- [x] **Report Completion** (`submission/REPORT.md`)
  - [x] Mục 1: Thông tin nhóm ✅
  - [x] Mục 2: Kết quả kỹ thuật ✅
  - [x] Mục 3: Logging và tracing ✅
  - [x] Mục 4: Prompt versioning ✅
  - [x] Mục 5: Dashboard, SLO và alerts ✅
  - [x] Mục 6: Điều tra challenge ✅
  - [x] Mục 7: Đóng góp cá nhân ✅

### Kết quả:
- [x] Challenge investigation hoàn thành theo M→T→L
- [x] REPORT.md: **7/7 sections complete**
- [x] Evidence files đầy đủ

### Evidence:
- [x] `submission/evidence/challenge_metrics_symptom.png`
- [x] `submission/evidence/challenge_trace_span.png`
- [x] `submission/evidence/challenge_log_rootcause.json`

### Git Commit:
```
docs(report): complete incident investigation and submission evidence
Author: Đinh Lê Quỳnh Phương
```

---

## ✅ CHECKPOINT 4: SUBMISSION & DEMO PREP

### Hoàn thành:
- [x] **Repository Status**
  - [x] All code pushed to `main` branch
  - [x] Latest commit SHA: `e0bbe6b`
  - [x] Repository URL: https://github.com/quynhphuong1209/Day13-K3-C2-3
  - [x] All commits có author name khớp với REPORT.md

- [x] **Evidence Files** (10 files total)
  - [x] `correlation_id_log.json`
  - [x] `pii_redact_log.json`
  - [x] `validate_logs.png`
  - [x] `langfuse_traces.png`
  - [x] `trace_waterfall.png`
  - [x] `prompt_v1_v2.png`
  - [x] `prompt_rollback.png`
  - [x] `dashboard.png`
  - [x] `validate_dashboard.png`
  - [x] `challenge_log_rootcause.json`

- [x] **Documentation**
  - [x] `DEMO_SCRIPT.md`: Script demo 10 phút
  - [x] `PRACTICE_QUESTIONS.md`: Câu hỏi luyện tập cho từng thành viên
  - [x] `TEAM_ROLES.md`: Phân công vai trò chi tiết
  - [x] `submission/REPORT.md`: Báo cáo hoàn chỉnh

- [x] **Validation Scripts**
  - [x] `python scripts/validate_logs.py` → 100/100 ✅
  - [x] `python scripts/validate_dashboard.py` → 6/6 panels ✅
  - [x] `python -m pytest -q` → All tests pass ✅

---

## 🎯 CHUẨN BỊ DEMO (A3 - 20 điểm)

### Pre-demo checklist (30 phút trước):
- [ ] Test `uvicorn app.main:app --reload` → server start OK
- [ ] Test `python scripts/load_test.py` → logs generated
- [ ] Verify `data/logs.jsonl` có 21 clean logs
- [ ] Open Langfuse dashboard → confirm ≥10 traces visible
- [ ] Open 2 terminals: T1 (uvicorn), T2 (scripts)
- [ ] Open files: `DEMO_SCRIPT.md`, `docs/dashboard-spec.md`, `submission/evidence/`

### Demo flow (10 phút):
- [ ] **Bước 1** (1 phút): Khởi động hệ thống - Thành viên C
- [ ] **Bước 2** (3 phút): CP1 Logging demo - Thành viên A
  - [ ] Show correlation_id format `req-XXXXXXXX`
  - [ ] Show context enrichment (user_id_hash, session_id, feature, model, env)
  - [ ] Show PII scrubbing ([REDACTED_EMAIL], [REDACTED_PHONE_VN])
  - [ ] Run `validate_logs.py` → 100/100
- [ ] **Bước 3** (4 phút): CP2 Metrics & Traces - Thành viên B
  - [ ] Metrics endpoint: show 6 metrics groups
  - [ ] Dashboard spec: 6 panels with SLO lines
  - [ ] Langfuse traces: waterfall + correlation_id in metadata
  - [ ] Prompt versioning: v1 vs v2, rollback
- [ ] **Bước 4** (2 phút): CP3 Investigation - Thành viên C
  - [ ] M→T→L flow: Metrics (P95=2750ms) → Traces (span retrieve) → Logs (req-c1a2b3d4)
  - [ ] Root cause: `rag_slow` incident in `retrieve()` for feature="refund"
  - [ ] Fix & Preventive measures

### Backup plan nếu lỗi:
- [ ] Nếu server không start → chỉ evidence screenshots
- [ ] Nếu Langfuse lỗi → dùng `submission/evidence/` screenshots
- [ ] Nếu scripts lỗi → explain bằng logs.jsonl có sẵn

---

## 🎓 CHUẨN BỊ TRÁ LỜI (B1 - 20 điểm)

### Thành viên A - Đoàn Minh Hiếu:
- [ ] Giải thích `clear_contextvars()` tại sao bắt buộc
- [ ] So sánh log baseline (30/100) vs CP1 (100/100)
- [ ] Tại sao hash user_id thay vì log trực tiếp
- [ ] PII scrubbing đệ quy hoạt động thế nào
- [ ] Generic exception handler mục đích gì

### Thành viên B - Kim Mạnh Hưng:
- [ ] Symptom-based vs Cause-based alerts
- [ ] Giải thích P50/P95/P99 percentiles
- [ ] Tại sao cần dashboard khi đã có /metrics endpoint
- [ ] Prompt versioning giải quyết vấn đề gì
- [ ] Correlation ID xuất hiện trong trace metadata như thế nào

### Thành viên C - Đinh Lê Quỳnh Phương:
- [ ] Giải thích luồng M→T→L investigation
- [ ] Nếu chỉ có metrics, điều tra khó khăn thế nào
- [ ] Fix action vs Preventive measure khác nhau gì
- [ ] Correlation ID giúp gì trong incident investigation
- [ ] Tại sao không đi theo thứ tự L→T→M

### Câu hỏi chung (cả nhóm):
- [ ] Ba trụ cột Observability là gì
- [ ] Tại sao cần Correlation ID
- [ ] Tại sao phải scrub PII trước khi log

---

## 📚 TÀI LIỆU THAM KHẢO

### Files quan trọng cần review:
1. **DEMO_SCRIPT.md** - Script demo chi tiết 10 phút
2. **PRACTICE_QUESTIONS.md** - Bộ câu hỏi luyện tập đầy đủ
3. **submission/REPORT.md** - Báo cáo chính thức nộp bài
4. **TEAM_ROLES.md** - Phân công vai trò và nhiệm vụ

### Evidence screenshots:
- Tất cả 10 files trong `submission/evidence/`
- Mở sẵn để demo nhanh nếu live demo gặp lỗi

### Commands cần nhớ:
```bash
# Start server
uvicorn app.main:app --reload --env-file .env

# Load test
python scripts/load_test.py

# Validation
python scripts/validate_logs.py
python scripts/validate_dashboard.py

# Metrics
curl http://localhost:8000/metrics | python -m json.tool

# View logs
tail -f data/logs.jsonl
```

---

## 🏆 MỤC TIÊU CUỐI CÙNG

### Điểm số kỳ vọng:
- **A1**: 30/30 ✅ (đã đạt)
- **A2**: 10/10 ✅ (đã đạt)
- **A3**: 18-20/20 🎯 (demo tự tin + giải thích rõ)
- **B1**: 18-20/20 🎯 (trả lời đúng trọng tâm)
- **B2**: 20/20 ✅ (đã đạt)

### **TỔNG: 96-100/100** 🏆

---

## ✅ STATUS: SẴN SÀNG 100%

Tất cả code implementation, evidence files, documentation đã hoàn thành.
Nhóm chỉ cần:
1. ✅ Luyện tập demo script 2-3 lần
2. ✅ Review câu hỏi trong PRACTICE_QUESTIONS.md
3. ✅ Tự tin trình bày trong 10-12 phút
4. ✅ Trả lời câu hỏi phản biện rõ ràng

**🎯 MỤC TIÊU: 100/100 ĐIỂM TUYỆT ĐỐI**

**Chúc cả nhóm thành công! 🎉🏆**
