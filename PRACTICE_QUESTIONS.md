# Bộ Câu hỏi Luyện tập Demo Lab 13 - Nhóm C2-3 (K3)
## Chuẩn bị cho Phần A3 (Demo) và B1 (Hiểu bài cá nhân)

---

## 🎯 CÂU HỎI CHUNG (Tất cả thành viên cần biết)

### 1. Ba trụ cột Observability là gì? Vai trò của từng trụ cột?
**Trả lời:**
- **Metrics**: Phát hiện triệu chứng (What) - Chỉ số tổng hợp như latency P95, error rate, cost
- **Traces**: Khoanh vùng vị trí (Where) - Waterfall timeline của request, thấy span nào chậm
- **Logs**: Giải thích nguyên nhân (Why) - Chi tiết từng sự kiện, exception, error message

### 2. Tại sao cần Correlation ID?
**Trả lời:**
Correlation ID (format `req-<8hex>`) cho phép:
- Trace request xuyên suốt toàn bộ hệ thống (API → RAG → LLM → Database)
- Đối chiếu được giữa Metrics, Traces và Logs
- Debug nhanh: từ trace ID → tìm logs liên quan → phát hiện root cause
- Không có correlation ID → logs rời rạc, không biết log nào thuộc request nào

### 3. Tại sao phải scrub PII trước khi log?
**Trả lời:**
- **Compliance**: GDPR, PDPA yêu cầu bảo vệ dữ liệu cá nhân
- **Security**: Logs thường được lưu trữ lâu dài, dễ bị leak
- **Best practice**: Scrub PII ở processor layer → đảm bảo không PII nào rò rỉ ra log file, monitoring system, hoặc SIEM

---

## 👤 THÀNH VIÊN A - Đoàn Minh Hiếu (CP1 - Logging)

### Câu hỏi kỹ thuật:

#### 1. Giải thích luồng xử lý correlation ID trong middleware?
**Trả lời:**
```python
# Bước 1: Clear contextvars cũ (tránh data leakage)
clear_contextvars()

# Bước 2: Lấy/sinh correlation_id
correlation_id = request.headers.get("x-request-id") or f"req-{uuid.uuid4().hex[:8]}"

# Bước 3: Bind vào contextvars (cho structlog) và request.state (cho handlers)
bind_contextvars(correlation_id=correlation_id)
request.state.correlation_id = correlation_id

# Bước 4: Trả về header response
response.headers["x-request-id"] = correlation_id
```

#### 2. Context enrichment hoạt động như thế nào?
**Trả lời:**
Trong endpoint `/chat`, gọi `bind_contextvars()` với metadata:
- `user_id_hash`: SHA-256 hash của user_id (bảo vệ PII)
- `session_id`, `feature`, `model`, `env`: Metadata từ request payload
- Contextvars tự động lan truyền xuống mọi log call trong cùng request context
- Không cần pass qua từng hàm → clean code

#### 3. PII scrubbing processor hoạt động ở layer nào?
**Trả lời:**
Ở processor chain của structlog (trong `logging_config.py`):
```python
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.TimeStamper(fmt="iso"),
        scrub_event,  # ← PII scrubbing ở đây (trước JSONRenderer)
        structlog.processors.JSONRenderer(),
    ]
)
```
Scrub TRƯỚC khi render JSON → đảm bảo output JSON không có PII

#### 4. Tại sao phải scrub đệ quy (recursive)?
**Trả lời:**
Event dict có thể chứa nested structure:
```python
{
  "event": "request_received",
  "payload": {
    "message": "My email is john@example.com",  # ← Cần scrub
    "metadata": {
      "user_contact": "+84901234567"  # ← Cũng cần scrub
    }
  }
}
```
Scrubbing đệ quy duyệt toàn bộ dict và sub-dict → không bỏ sót PII

#### 5. Generic Exception Handler mục đích gì?
**Trả lời:**
Khi xảy ra exception 500, middleware có thể bị bypass → response thiếu `x-request-id`.
Generic Exception Handler đảm bảo:
- Mọi response (kể cả lỗi) đều có `x-request-id` header
- Client có thể báo correlation_id khi report bug
- Team debug nhanh bằng cách grep logs với correlation_id đó

---

## 👤 THÀNH VIÊN B - Kim Mạnh Hưng (CP2 - Tracing & Dashboard)

### Câu hỏi kỹ thuật:

#### 1. Làm thế nào để correlation_id xuất hiện trong Langfuse trace?
**Trả lời:**
Trong `app/agent.py`, lấy contextvars và gắn vào trace metadata:
```python
from structlog.contextvars import get_contextvars

correlation_id = get_contextvars().get("correlation_id")
trace_metadata = {"correlation_id": correlation_id}

@observe(as_type="generation", metadata=trace_metadata)
def run(...):
    ...
```
→ Trace trên Langfuse UI sẽ hiển thị correlation_id trong metadata panel

#### 2. Prompt versioning giải quyết vấn đề gì?
**Trả lời:**
- **A/B testing**: So sánh prompt v1 (baseline) vs v2 (candidate) về quality, latency, cost
- **Rollback**: Nếu v2 làm quality giảm → đổi label `production` về v1 ngay lập tức
- **Audit trail**: Biết trace nào dùng prompt version nào
- **Collaboration**: Team cùng nhìn thấy prompt history và changes

#### 3. Tại sao cần thêm error_rate_pct vào metrics?
**Trả lời:**
Error rate % cho biết:
- Tỷ lệ request thất bại (timeout, exception, tool_fail)
- SLO thường yêu cầu error rate < 2%
- Tăng đột biến error rate → incident nghiêm trọng (user không dùng được service)
- Formula: `error_rate_pct = (total_errors / total_requests) * 100`

#### 4. Dashboard 6 panels phục vụ mục đích gì?
**Trả lời:**
1. **Latency Percentiles**: Theo dõi trải nghiệm user (P50/P95/P99)
2. **Traffic**: Capacity planning (QPS tăng → cần scale)
3. **Error Rate**: Reliability (SLO commitment)
4. **Cost**: Budget control (tránh surprise bill)
5. **Tokens**: Optimize prompt (reduce input tokens → giảm cost)
6. **Quality**: Model performance (quality giảm → cần retrain/tune prompt)

#### 5. Symptom-based alert vs Cause-based alert?
**Trả lời:**
| | Symptom-based | Cause-based |
|---|---|---|
| **Trigger** | Latency P95 > 3s | Function `retrieve()` timeout |
| **Pros** | Luôn bắt được user impact, không bị outdated khi code thay đổi | Cụ thể, dễ debug |
| **Cons** | Cần drill down để tìm root cause | Dễ bị vô hiệu khi refactor code, miss các nguyên nhân khác |
| **Production** | ✅ Recommended | ❌ Avoid |

#### 6. Alert runbook cần có gì?
**Trả lời:**
5 phần bắt buộc:
1. **Severity & Condition**: P1/P2/P3, trigger threshold
2. **User Impact**: Người dùng thấy gì? (slow response, error 500, ...)
3. **3 bước kiểm tra đầu tiên**: M→T→L hoặc check service health
4. **Mitigation**: Fix tạm thời (scale up, circuit breaker, rollback)
5. **Owner**: Team/person on-call phụ trách

---

## 👤 THÀNH VIÊN C - Đinh Lê Quỳnh Phương (CP3 - Incident Investigation)

### Câu hỏi kỹ thuật:

#### 1. Tại sao phải đi theo thứ tự M→T→L (không phải L→T→M)?
**Trả lời:**
- **Logs**: Hàng ngàn dòng/phút, quá nhiễu, không biết tìm gì
- **Traces**: Chỉ thấy request cụ thể, không biết bao nhiêu request bị ảnh hưởng
- **Metrics**: Big picture, phát hiện anomaly (latency spike, error rate jump)

Quy trình đúng:
1. Metrics phát hiện triệu chứng → biết "có vấn đề"
2. Traces khoanh vùng → biết "vấn đề ở đâu" (span nào, feature nào)
3. Logs chứng minh → biết "tại sao" (error message, exception stack)

#### 2. Trong challenge rag_slow, làm sao biết feature="refund" bị ảnh hưởng?
**Trả lời:**
Bước 1: Metrics → thấy latency P95 = 2750ms
Bước 2: Traces → lọc traces có latency > 2s, xem metadata → thấy `feature: "refund"`
Bước 3: Grep logs với `feature: "refund"` → confirm tất cả request refund đều chậm

#### 3. Nếu không có traces, điều tra sẽ khó thế nào?
**Trả lời:**
Không có traces:
- Phải đọc hết logs để tìm request chậm (hàng ngàn dòng)
- Không biết bước nào trong pipeline bị chậm (RAG? LLM? Database?)
- Phải instrument code thêm timer → deploy lại → chờ reproduce
- Tốn nhiều giờ thay vì vài phút

Có traces:
- Waterfall UI chỉ rõ: span `retrieve` chiếm 2.5s / 2.75s total
- Ngay lập tức biết RAG là bottleneck

#### 4. Fix action vs Preventive measure - ví dụ cụ thể?
**Trả lời:**
**Incident: RAG retrieval chậm 2.5s**

Fix action (giải quyết ngay):
- Restart RAG service
- Tăng timeout cho retrieve() từ 5s → 10s (tạm thời)
- Enable cache cho queries thường gặp

Preventive measure (ngăn tái diễn):
- Optimize vector store index (HNSW → IVF-PQ)
- Alert span-level: `retrieve > 500ms` → cảnh báo sớm
- Performance test trong CI: fail build nếu retrieve > 300ms
- APM dashboard cho RAG service riêng

#### 5. Correlation ID giúp gì trong điều tra incident?
**Trả lời:**
**Scenario**: User báo "request bị lỗi lúc 14:35"

Không có correlation ID:
- Grep logs theo timestamp → 50 requests cùng lúc
- Không biết request nào của user đó
- Phải hỏi user thêm thông tin (session_id, message, ...)

Có correlation ID:
- User cung cấp `x-request-id: req-abc12345` từ HTTP response header
- Grep logs: `correlation_id: "req-abc12345"` → tìm ngay dòng log lỗi
- Tìm trace trên Langfuse với metadata `correlation_id: "req-abc12345"`
- Debug chính xác request đó trong < 1 phút

---

## 🎤 CÂU HỎI PHẢN BIỆN NÂNG CAO

### Câu 1: Nếu Coach hỏi "Tại sao không dùng OpenTelemetry thay vì Langfuse?"
**Trả lời:**
- **OpenTelemetry**: Standard protocol, vendor-agnostic, production-grade
- **Langfuse**: LLM-specific platform, có prompt management, quality tracking
- Trong lab này chọn Langfuse vì:
  - Prompt versioning built-in
  - UI đẹp, dễ demo
  - Tích hợp nhanh với LangChain/LlamaIndex
- Production thực tế: có thể dùng cả hai (OTel export sang Langfuse)

### Câu 2: "Logs.jsonl có 21 dòng, production có 10 triệu request/ngày thì sao?"
**Trả lời:**
Production cần:
- **Log aggregation**: Ship logs sang Elasticsearch, Datadog, CloudWatch Logs
- **Retention policy**: Chỉ giữ 7-30 ngày, sau đó archive sang S3
- **Sampling**: Log 100% errors, chỉ sample 1-10% success requests
- **Indexing**: Index theo correlation_id, user_id_hash, feature → query nhanh

### Câu 3: "Nếu Langfuse down, traces bị mất?"
**Trả lời:**
- Langfuse SDK có **buffering** và **retry** logic
- Nếu Langfuse down tạm thời → traces được buffer, gửi lại khi Langfuse up
- Nếu down lâu → traces bị drop, nhưng **Metrics và Logs vẫn hoạt động**
- Mitigation: Self-host Langfuse hoặc export traces sang backup (OTel collector)

### Câu 4: "Alert false positive (kêu oan) xử lý thế nào?"
**Trả lời:**
Nguyên nhân false positive:
- Threshold quá thấp (alert khi P95 > 1s nhưng baseline là 800ms)
- Không có context (deploy mới → latency tăng tạm thời 2 phút là bình thường)

Giải pháp:
- **Điều chỉnh threshold** dựa trên baseline thực tế
- **Alert với duration**: chỉ kêu khi P95 > 3s **trong 5 phút liên tục** (tránh spike thoáng qua)
- **Correlation**: alert khi (latency tăng AND error rate tăng) → chắc chắn hơn

### Câu 5: "SLO 99.5% requests có P95 < 3s nghĩa là gì?"
**Trả lời:**
Ví dụ 1 tháng có 1 triệu requests:
- **99.5% tháng** = 30 ngày × 0.995 = 29.85 ngày OK, 0.15 ngày (3.6 giờ) được phép vi phạm
- Trong mỗi ngày OK: **P95 < 3s** = 95% requests nhanh hơn 3s (5% chậm hơn vẫn OK)

Tính toán error budget:
- Total request: 1,000,000
- Allowed violations: 1,000,000 × (1 - 0.995) = 5,000 requests
- Nếu đã vi phạm 5,000 requests → dừng deploy feature mới, focus stability

---

## ✅ CHECKLIST CHUẨN BỊ DEMO

### Trước demo (30 phút):
- [ ] Test `uvicorn app.main:app --reload` chạy thành công
- [ ] Test `python scripts/load_test.py` sinh logs OK
- [ ] Test `python scripts/validate_logs.py` → 100/100
- [ ] Test `python scripts/validate_dashboard.py` → 6/6 panels
- [ ] Mở Langfuse dashboard, kiểm tra có >= 10 traces
- [ ] Mở sẵn các file: `DEMO_SCRIPT.md`, `docs/dashboard-spec.md`, `submission/evidence/`
- [ ] Mở 2 terminal: T1 (uvicorn), T2 (scripts)

### Trong demo:
- [ ] Nói rõ ràng, không nói quá nhanh
- [ ] Chỉ rõ file path, line number khi giải thích code
- [ ] Demo từng checkpoint tuần tự: CP1 → CP2 → CP3
- [ ] Highlight kết quả: 100/100 logs, 6/6 dashboard, M→T→L investigation

### Khi trả lời câu hỏi:
- [ ] Nghe hết câu hỏi, không ngắt lời giám khảo
- [ ] Suy nghĩ 2-3 giây trước khi trả lời
- [ ] Trả lời theo structure: Khái niệm → Lý do → Ví dụ
- [ ] Nếu không biết: "Em chưa nắm rõ phần này, nhưng em nghĩ..."
- [ ] Không tranh luận, không nói "không phải vậy"

---

## 🎯 KỲ VỌNG ĐIỂM

| Phần | Điểm tối đa | Kỳ vọng | Ghi chú |
|---|---|---|---|
| A1 (Kỹ thuật) | 30 | 30/30 ✅ | Đã hoàn thành code |
| A2 (Investigation) | 10 | 10/10 ✅ | REPORT.md đầy đủ |
| A3 (Demo) | 20 | 18-20 | Phụ thuộc presentation |
| B1 (Hiểu bài) | 20 | 18-20 | Phụ thuộc trả lời câu hỏi |
| B2 (Git commits) | 20 | 20/20 ✅ | Commits đã khớp REPORT.md |
| **Tổng** | **100** | **96-100** | 🏆 Mục tiêu: 100/100 |

**Lưu ý**: Với code implementation 100% hoàn thiện, nhóm chỉ cần demo tự tin và trả lời câu hỏi rõ ràng để đạt **100/100 điểm tuyệt đối**.

---

**Chúc cả nhóm tự tin, thành công và đạt điểm tối đa! 🎯🏆**
