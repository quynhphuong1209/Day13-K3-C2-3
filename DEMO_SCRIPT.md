# Script Demo Lab 13 - Nhóm C2-3 (K3)
## Mục tiêu: Đạt 20/20 điểm A3 (Demo & Giải thích)

---

## 🎯 PHẦN 1: DEMO TRỰC TIẾP (10 phút)

### Bước 1: Khởi động hệ thống (1 phút)
```bash
# Terminal 1 - Start API server
uvicorn app.main:app --reload --env-file .env

# Chờ server khởi động, sau đó Terminal 2
```

**Người demo:** Thành viên C (Đinh Lê Quỳnh Phương)
**Nói:** "Chúng em sẽ demo hệ thống AI API đã được tích hợp Observability đầy đủ với logging, metrics, traces và alerts."

---

### Bước 2: Demo CP1 - Logging với Correlation ID & PII (3 phút)

**Người demo:** Thành viên A (Đoàn Minh Hiếu)

```bash
# Terminal 2 - Gửi request
python scripts/load_test.py

# Xem logs real-time
tail -f data/logs.jsonl
```

**Giải thích trong khi chạy:**
1. **Correlation ID:**
   - "Mỗi request có ID duy nhất format `req-<8hex>` để trace xuyên suốt"
   - "Chỉ log có correlation_id: `req-990c4a75`"
   - "Header response cũng trả về x-request-id để client track được"

2. **Context Enrichment:**
   - "Mọi log tự động có: user_id_hash, session_id, feature, model, env"
   - "Không cần pass qua từng hàm, dùng contextvars"

3. **PII Scrubbing:**
   - "Email → [REDACTED_EMAIL]"
   - "Phone VN → [REDACTED_PHONE_VN]"
   - "Passport, Address VN đều được che"
   - "Scrubbing đệ quy toàn bộ event dict, không bỏ sót"

```bash
# Validation
python scripts/validate_logs.py
# → Kết quả: 100/100
```

**Highlight:** "Điểm số 100/100 chứng minh logs của chúng em đã đạt chuẩn production: có correlation ID, có enrichment đầy đủ, không lộ PII."

---

### Bước 3: Demo CP2 - Metrics & Traces (4 phút)

**Người demo:** Thành viên B (Kim Mạnh Hưng)

#### 3a. Metrics Endpoint (1 phút)
```bash
curl http://localhost:8000/metrics | python -m json.tool
```

**Giải thích từng chỉ số:**
```json
{
  "traffic": 10,                    // ← Số lượng request
  "latency_p50": 1150,              // ← 50% request < 1.15s
  "latency_p95": 2184,              // ← 95% request < 2.18s
  "latency_p99": 2184,              // ← 99% request < 2.18s
  "error_rate_pct": 0.0,            // ← Tỷ lệ lỗi 0%
  "total_cost_usd": 0.0183,         // ← Chi phí tích lũy
  "quality_avg": 0.9                // ← Điểm chất lượng trung bình
}
```

**Nói:** "Metrics này là nguồn dữ liệu cho dashboard và alert rules. Chúng em theo dõi 6 nhóm chỉ số: Latency, Traffic, Error, Cost, Tokens, Quality."

#### 3b. Dashboard (1 phút)
**Mở file:** `docs/dashboard-spec.md`

**Nói:** "Dashboard của chúng em có 6 panels:"
1. **Latency Percentiles** - P50/P95/P99 với SLO line ≤3000ms
2. **Traffic** - Tổng request và QPS
3. **Error Rate** - Tỷ lệ lỗi % với threshold ≤2%
4. **Cost** - Budget tracking ≤$2.5/day
5. **Tokens** - Input/output token usage
6. **Quality** - Điểm trung bình ≥0.75

```bash
python scripts/validate_dashboard.py
# → HỢP LỆ: 6/6 panel
```

#### 3c. Langfuse Traces (2 phút)
**Mở browser:** Langfuse Dashboard

**Nói:** "Traces trên Langfuse cho phép chúng em nhìn thấy waterfall timeline của mỗi request."

**Chỉ vào evidence screenshot:**
- `langfuse_traces.png` - "Danh sách 10+ traces"
- `trace_waterfall.png` - "Chi tiết 1 trace: span run, correlation_id trong metadata"
- `prompt_v1_v2.png` - "Prompt versioning: baseline vs candidate"
- `prompt_rollback.png` - "Rollback từ v2 về v1 khi cần"

**Highlight:** "Correlation ID được đính kèm vào trace metadata → đối chiếu được giữa Logs và Traces."

---

### Bước 4: Demo CP3 - Incident Investigation (2 phút)

**Người demo:** Thành viên C (Đinh Lê Quỳnh Phương)

**Kịch bản:** "Giả lập incident `rag_slow` đã được Coach release"

```bash
# Bật incident
python scripts/inject_incident.py

# Chạy challenge queries
python scripts/load_test.py --challenge --concurrency 5
```

**Quy trình điều tra M→T→L:**

1. **Metrics** - Phát hiện triệu chứng
```bash
curl http://localhost:8000/metrics | python -m json.tool
```
**Nói:** "Latency P95 tăng từ ~300ms lên 2750ms - vượt ngưỡng SLO 2000ms"

2. **Traces** - Khoanh vùng vị trí
**Chỉ vào:** `submission/evidence/challenge_trace_span.png`
**Nói:** "Trace waterfall chỉ rõ span `retrieve` kéo dài 2.5s"

3. **Logs** - Chứng minh root cause
**Mở file:** `submission/evidence/challenge_log_rootcause.json`
```json
{
  "correlation_id": "req-c1a2b3d4",
  "latency_ms": 2752,
  "feature": "refund",
  "event": "response_sent"
}
```

**Nói:** "Root cause: Hàm `retrieve()` trong mock_rag.py bị delay 2.5s khi incident `rag_slow` active cho feature `refund`."

**Fix action:** "Tối ưu vector store, thêm timeout 500ms, circuit breaker fallback"

**Preventive:** "Alert cho span-level latency, APM monitoring, regression tests"

---

## 🎯 PHẦN 2: CÂU HỎI PHẢN BIỆN (Điểm B1 - 20 điểm)

### **Thành viên A - Đoàn Minh Hiếu (CP1)**

#### Câu 1: Tại sao phải gọi `clear_contextvars()` ở đầu middleware?
**Trả lời:**
"Trong FastAPI async, một thread/task có thể được dùng lại để xử lý nhiều request khác nhau. Nếu không clear contextvars ở đầu mỗi request mới, contextvars của request cũ sẽ bị 'dính' sang request mới, gây rò rỉ dữ liệu (data leakage) giữa các user. Ví dụ: user A gửi request với correlation_id=req-aaa, sau đó user B gửi request nhưng log lại ghi correlation_id=req-aaa → nhầm lẫn nghiêm trọng."

#### Câu 2: So sánh log baseline (CP0) và log sau CP1?
**Trả lời:**
"Log baseline (CP0):
- Không có correlation_id (hiển thị 'MISSING')
- Không có enrichment context (thiếu user_id_hash, session_id, feature, model, env)
- PII chỉ được che trong message_preview, chưa che toàn bộ event

Log sau CP1:
- Có correlation_id format `req-<8hex>` cho mọi request
- Có đầy đủ enrichment context tự động
- PII được scrub đệ quy toàn bộ event dict
- Điểm validator: 30/100 → 100/100"

#### Câu 3: Tại sao hash user_id thay vì log trực tiếp?
**Trả lời:**
"User ID là PII - Personally Identifiable Information. Log trực tiếp vi phạm quy định bảo vệ dữ liệu cá nhân (GDPR, PDPA). Dùng SHA-256 hash cho phép:
- Vẫn track được user qua các request (cùng user → cùng hash)
- Không thể reverse engineer để lấy user_id gốc
- Tuân thủ compliance"

---

### **Thành viên B - Kim Mạnh Hưng (CP2)**

#### Câu 1: Tại sao alert rules phải là Symptom-based thay vì Cause-based?
**Trả lời:**
"Symptom-based alerts cảnh báo dựa trên triệu chứng người dùng thấy (latency cao, error rate tăng), không phụ thuộc vào implementation cụ thể. Ví dụ:
- ❌ Cause-based: 'Alert khi hàm retrieve() timeout' → Nếu đổi tên hàm hoặc chuyển sang RAG provider khác, alert bị vô hiệu
- ✅ Symptom-based: 'Alert khi latency P95 > 3s' → Luôn hoạt động dù code thay đổi, bắt được mọi nguyên nhân gây chậm

Symptom-based đảm bảo không bỏ sót sự cố thực tế và giảm alert noise."

#### Câu 2: Giải thích ý nghĩa P50, P95, P99?
**Trả lời:**
"Percentile là phân vị:
- P50 (median): 50% request nhanh hơn giá trị này → đại diện cho trải nghiệm 'typical user'
- P95: 95% request nhanh hơn → 5% còn lại là 'tail latency', phản ánh worst-case
- P99: 99% request nhanh hơn → 1% extreme outliers

Tại sao theo dõi P95/P99 chứ không chỉ average?
- Average bị ảnh hưởng bởi outliers, che giấu vấn đề
- P95/P99 phát hiện vấn đề tail latency ảnh hưởng đến một phần user
- SLO production thường dùng P95: '99.5% request có P95 < 3s'"

#### Câu 3: Tại sao cần dashboard khi đã có metrics endpoint?
**Trả lời:**
"Metrics endpoint `/metrics` trả raw numbers, khó nhìn ra pattern và trend. Dashboard:
- Visualize time-series: nhìn thấy latency tăng dần hay tăng đột biến
- So sánh nhiều chỉ số cùng lúc: latency tăng + error rate tăng → incident nghiêm trọng
- SLO line: vạch ngưỡng cam kết, dễ nhìn khi vượt ngưỡng
- Alert context: khi alert kêu, xem dashboard để understand big picture"

---

### **Thành viên C - Đinh Lê Quỳnh Phương (CP3)**

#### Câu 1: Giải thích luồng điều tra Metrics → Traces → Logs?
**Trả lời:**
"Luồng điều tra đi từ tổng quan đến chi tiết:

**Bước 1 - Metrics (Phát hiện):**
- Nhìn `/metrics` thấy latency P95 vọt lên 2750ms (vượt SLO 2000ms)
- Error rate vẫn 0% → request thành công nhưng chậm
- → Triệu chứng: Latency spike

**Bước 2 - Traces (Khoanh vùng):**
- Mở Langfuse, lọc traces trong khoảng thời gian bất thường
- Xem waterfall: span `run` kéo dài 2.75s
- Sub-span `retrieve` chiếm 2.5s
- Metadata shows feature='refund'
- → Vị trí: RAG retrieval của feature 'refund'

**Bước 3 - Logs (Chứng minh):**
- Lấy correlation_id từ trace: req-c1a2b3d4
- Grep logs: tìm log có cùng correlation_id
- Thấy log `response_sent` với latency_ms=2752, feature='refund'
- → Root cause: retrieve() bị delay 2.5s khi incident rag_slow active

Không đi ngược lại vì:
- Logs quá chi tiết, khó tìm pattern trong hàng ngàn dòng
- Traces không có context tổng thể (có bao nhiêu request bị ảnh hưởng?)
- Metrics cho big picture trước → drill down từ từ"

#### Câu 2: Nếu hệ thống chỉ có metrics, điều tra sẽ khó khăn như thế nào?
**Trả lời:**
"Chỉ có metrics thì:
- Biết 'có sự cố' nhưng không biết 'ở đâu': latency tăng → do RAG? LLM? Network?
- Không khoanh vùng được feature hoặc user segment nào bị ảnh hưởng
- Phải đọc code hoặc restart toàn bộ service → downtime lâu
- Không có correlation ID → không trace được request cụ thể

Ba trụ cột phải đi cùng nhau:
- Metrics: What (triệu chứng)
- Traces: Where (vị trí)
- Logs: Why (nguyên nhân)"

#### Câu 3: Fix action và Preventive measure khác nhau thế nào?
**Trả lời:**
"**Fix action:** Giải quyết sự cố hiện tại
- Tối ưu chỉ mục vector store → giảm latency retrieval
- Thêm timeout 500ms cho retrieve() → fail fast thay vì chờ lâu
- Circuit breaker fallback về cache → vẫn trả response khi RAG chậm

**Preventive measure:** Ngăn sự cố lặp lại trong tương lai
- Alert span-level: retrieve > 500ms → cảnh báo sớm trước khi ảnh hưởng P95
- APM monitoring cho vector database → track response time liên tục
- Performance regression tests trong CI/CD → phát hiện code làm chậm hệ thống trước khi deploy production

Fix action là 'chữa bệnh', Preventive là 'phòng bệnh'."

---

## 🎯 PHẦN 3: CHECKLIST ĐẠT 100/100

### A3 - Demo và giải thích (20 điểm)

**Để đạt 20/20:**
- [ ] Hệ thống chạy mượt mà, không lỗi
- [ ] Demo đủ 3 checkpoint: CP1 (logs), CP2 (metrics/traces), CP3 (investigation)
- [ ] Giải thích rõ ràng từng bước, không đọc slides
- [ ] Trả lời được câu hỏi "Tại sao làm vậy?" chứ không chỉ "Làm gì?"
- [ ] Liên hệ được với real-world production scenarios
- [ ] Chỉ được evidence cụ thể (screenshot, logs, metrics)
- [ ] Thời gian demo 10-12 phút (không quá 15 phút)

### B1 - Hiểu bài cá nhân (20 điểm)

**Để đạt 20/20:**
- [ ] Mỗi thành viên trả lời thành thạo phần việc của mình
- [ ] Hiểu WHY (lý do thiết kế) chứ không chỉ WHAT (làm gì)
- [ ] Biết trade-offs: tại sao chọn cách này thay vì cách khác
- [ ] Liên hệ với khái niệm Observability: 3 trụ cột, luồng M→T→L
- [ ] Trả lời ngắn gọn, đúng trọng tâm (30-60 giây/câu)
- [ ] Tự tin, không nói "em không biết" hoặc "bạn X làm phần này"

### B2 - Bằng chứng đóng góp (20 điểm)

**Đã đạt 20/20:**
- [x] Git commits rõ ràng với message mô tả đúng
- [x] Commit author khớp với tên trong REPORT.md
- [x] Phân công trong REPORT.md khớp với code thực tế
- [x] Mỗi thành viên có ít nhất 1 commit chính

---

## 📝 TIPS ĐẠT ĐIỂM TỐI ĐA

### Tips chung:
1. **Tự tin nhưng không ngạo:** "Chúng em đã làm X và đạt kết quả Y" (không nói "làm tốt nhất", "hoàn hảo")
2. **Cụ thể, không chung chung:** Nói con số, tên file, dòng code cụ thể
3. **Liên hệ production:** "Trong thực tế production, X sẽ giúp..."
4. **Thừa nhận limitation:** "Validator chỉ kiểm tra PII cơ bản, production cần thêm..."
5. **Chuẩn bị backup plan:** Nếu Langfuse lỗi, chỉ screenshot evidence

### Tips demo:
- Mở sẵn terminal, browser, files trước khi demo
- Test demo script 2-3 lần trước
- Nếu command chạy lâu, giải thích trong lúc chờ
- Có plan B: nếu lỗi, chỉ vào evidence screenshot

### Tips trả lời câu hỏi:
- Lắng nghe hết câu hỏi trước khi trả lời
- Trả lời theo cấu trúc: (1) Khái niệm, (2) Lý do, (3) Ví dụ
- Nếu không chắc: "Em nghĩ là X vì Y, nhưng có thể em hiểu chưa đầy đủ"
- Không tranh luận với giám khảo

---

## ✅ TỔNG KẾT

**Mục tiêu:** 100/100 điểm

**Hiện tại:**
- A1: 30/30 ✅
- A2: 10/10 ✅
- B2: 20/20 ✅

**Cần đạt thêm:**
- A3: 20/20 (demo script này)
- B1: 20/20 (câu hỏi phản biện trên)

**Công thức thành công:**
```
Demo tự tin + Giải thích rõ ràng + Trả lời đúng trọng tâm = 100/100
```

**Chúc cả nhóm đạt điểm tuyệt đối! 🎯🏆**
