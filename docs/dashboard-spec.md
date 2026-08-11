# Yêu cầu và Thiết kế Dashboard Spec
## Thành viên C (QA & Chief Incident Investigator) phụ trách

Contract chấm điểm tự động nằm tại `config/dashboard.yaml`. Hướng dẫn thiết kế và kiểm tra runtime nằm tại [DASHBOARD_SETUP.md](DASHBOARD_SETUP.md).

Dashboard chính gồm đầy đủ **6 nhóm panel chỉ số**:

---

### 1. Latency Percentiles (Panel ID: `latency`)
- **Nguồn dữ liệu**: `data/logs.jsonl` (Event: `response_sent`)
- **Chỉ số đo lường**: P50, P95, P99 của trường `latency_ms`.
- **Đơn vị**: `ms` (millisecond).
- **Threshold / SLO Line**: P95 ≤ 3000 ms.
- **Mục đích**: Phát hiện sự cố trễ mạng hoặc RAG/LLM bị trễ (Tail latency).

---

### 2. Request Traffic (Panel ID: `traffic`)
- **Nguồn dữ liệu**: `data/logs.jsonl` (Event: `request_received`)
- **Chỉ số đo lường**: Tổng số request (`count`) và lưu lượng request trên phút (`rate_per_minute`).
- **Đơn vị**: `requests_per_minute`.
- **Threshold / SLO Line**: QPS ≥ 1 request/phút.
- **Mục đích**: Giám sát tải hệ thống và lượng lưu lượng người dùng truy cập thực tế.

---

### 3. Error Rate & Breakdown (Panel ID: `errors`)
- **Nguồn dữ liệu**: `data/logs.jsonl` (Events: `request_received`, `request_failed`)
- **Chỉ số đo lường**: Tỷ lệ phần trăm lỗi (`error_rate_pct`) và phân rã số lượng theo loại lỗi (`error_type`).
- **Đơn vị**: `%` (percent).
- **Threshold / SLO Line**: `error_rate_pct` ≤ 2.0%.
- **Mục đích**: Cảnh báo tức thì khi hệ thống phát sinh lỗi bất thường (HTTP 500/timeout).

---

### 4. Cost Over Time (Panel ID: `cost`)
- **Nguồn dữ liệu**: `data/logs.jsonl` (Event: `response_sent`)
- **Chỉ số đo lường**: Tổng chi phí USD tích lũy (`sum(cost_usd)`) và chi phí theo phút.
- **Đơn vị**: `USD` ($).
- **Threshold / SLO Line**: Total Cost ≤ $2.50 / ngày.
- **Mục đích**: Quản lý ngân sách gọi API LLM, tránh tăng vọt chi phí ngoài dự kiến.

---

### 5. Input & Output Tokens (Panel ID: `tokens`)
- **Nguồn dữ liệu**: `data/logs.jsonl` (Event: `response_sent`)
- **Chỉ số đo lường**: Tổng số token nạp vào (`tokens_in`) và token sinh ra (`tokens_out`).
- **Đơn vị**: `tokens`.
- **Threshold / SLO Line**: Total Tokens ≤ 50,000 tokens.
- **Mục đích**: Theo dõi lượng token tiêu thụ của prompt và response LLM.

---

### 6. Quality Proxy (Panel ID: `quality`)
- **Nguồn dữ liệu**: `data/logs.jsonl` (Event: `response_sent`)
- **Chỉ số đo lường**: Điểm trung bình chất lượng câu trả lời (`mean(quality_score)`).
- **Đơn vị**: Thang điểm từ `0.0` đến `1.0`.
- **Threshold / SLO Line**: `quality_avg` ≥ 0.75.
- **Mục đích**: Giám sát proxy chất lượng câu trả lời từ mô hình AI.

---

### ⚙️ Lệnh kiểm tra contract tự động:

```bash
python scripts/validate_dashboard.py
```
