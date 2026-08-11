# Thiết kế Dashboard Observability cho Hệ thống AI (Dashboard Spec)

Tài liệu thiết kế chi tiết 6 nhóm chỉ số cho hệ thống AI Observability theo tiêu chuẩn contract quy định tại [`config/dashboard.yaml`](../config/dashboard.yaml).

---

## 1. Thông số cấu hình chung (General Configuration)

- **Công cụ sử dụng**: Grafana / Langfuse Dashboard / Custom Observability Dashboard (chạy dựa trên dữ liệu từ `/metrics` và `data/logs.jsonl`).
- **Nguồn dữ liệu chuẩn (Data Source)**: Endpoint `/metrics` & File log JSON `data/logs.jsonl`.
- **Khoảng thời gian mặc định (Time Range)**: `60 phút` (1 giờ gần nhất).
- **Tần suất làm mới (Auto Refresh)**: Mỗi `30 giây`.
- **Số lượng Panel chính**: 6 Panel chính tương ứng 6 nhóm chỉ số cốt lõi.

---

## 2. Thông tin chi tiết 6 Nhóm chỉ số (6 Dashboard Panels Spec)

### 📊 Panel 1: Độ trễ phản hồi (Latency Percentiles)
- **Tên Panel**: `Latency percentiles`
- **ID Panel**: `latency`
- **Nguồn dữ liệu**: `/metrics` (`latency_p50`, `latency_p95`, `latency_p99`) / `data/logs.jsonl` (event `response_sent`)
- **Trường dữ liệu**: `latency_ms`
- **Phép tổng hợp (Aggregation)**: Percentile P50, P95, P99
- **Đơn vị (Unit)**: `ms` (milliseconds)
- **Ngưỡng SLO / Threshold**: **P95 $\le$ 3000 ms** (Ngưỡng cảnh báo màu đỏ nếu P95 > 3000ms)
- **Dạng biểu đồ**: Line Chart (Biểu đồ đường theo thời gian hiển thị 3 đường P50, P95, P99).

### 📈 Panel 2: Lưu lượng truy cập (Request Traffic)
- **Tên Panel**: `Request traffic`
- **ID Panel**: `traffic`
- **Nguồn dữ liệu**: `/metrics` (`traffic`) / `data/logs.jsonl` (event `request_received`)
- **Trường dữ liệu**: `event`
- **Phép tổng hợp (Aggregation)**: `count() by 1m` (Request Count / phút)
- **Đơn vị (Unit)**: `requests_per_minute` (rpm)
- **Ngưỡng SLO / Threshold**: **Traffic $\ge$ 1 req/min** (Hệ thống ở trạng thái hoạt động bình thường)
- **Dạng biểu đồ**: Bar Chart / Area Chart (Biểu đồ cột/miền đếm lưu lượng request theo từng phút).

### 🚨 Panel 3: Tỷ lệ lỗi & Phân loại lỗi (Error Rate & Breakdown)
- **Tên Panel**: `Error rate and breakdown`
- **ID Panel**: `errors`
- **Nguồn dữ liệu**: `/metrics` (`error_rate_pct`, `error_breakdown`) / `data/logs.jsonl` (event `request_failed`)
- **Trường dữ liệu**: `error_type`
- **Phép tổng hợp (Aggregation)**: `error_rate_pct` (Tỷ lệ lỗi %) & `count_by(error_type)`
- **Đơn vị (Unit)**: `percent` (%)
- **Ngưỡng SLO / Threshold**: **Error Rate $\le$ 2%**
- **Dạng biểu đồ**: Stat Card (% tổng lỗi) kết hợp Donut/Pie Chart (Phân loại lỗi theo `LLMError`, `RAGError`, `Timeout`, v.v.).

### 💵 Panel 4: Chi phí sử dụng (Cost Over Time)
- **Tên Panel**: `Cost over time`
- **ID Panel**: `cost`
- **Nguồn dữ liệu**: `/metrics` (`total_cost_usd`, `avg_cost_usd`) / `data/logs.jsonl` (event `response_sent`)
- **Trường dữ liệu**: `cost_usd`
- **Phép tổng hợp (Aggregation)**: `sum(cost_usd) by 1m` & `sum(cost_usd)` tổng toàn cửa sổ
- **Đơn vị (Unit)**: `usd` ($)
- **Ngưỡng SLO / Threshold**: **Total Cost $\le$ 2.50 USD** / 60 phút
- **Dạng biểu đồ**: Line Chart (Tích luỹ chi phí theo phút) kèm Single Value Card đại diện cho tổng chi phí hiện tại.

### 🔢 Panel 5: Số lượng Token tiêu thụ (Input & Output Tokens)
- **Tên Panel**: `Input and output tokens`
- **ID Panel**: `tokens`
- **Nguồn dữ liệu**: `/metrics` (`tokens_in_total`, `tokens_out_total`) / `data/logs.jsonl` (event `response_sent`)
- **Trường dữ liệu**: `tokens_in`, `tokens_out`
- **Phép tổng hợp (Aggregation)**: `sum(tokens_in)`, `sum(tokens_out)`
- **Đơn vị (Unit)**: `tokens`
- **Ngưỡng SLO / Threshold**: **Total Tokens $\le$ 50,000 tokens** / 60 phút
- **Dạng biểu đồ**: Stacked Bar Chart (Phân biệt màu sắc giữa Input Token - Prompt và Output Token - Completion).

### ⭐ Panel 6: Chất lượng câu trả lời (Quality Proxy)
- **Tên Panel**: `Quality proxy`
- **ID Panel**: `quality`
- **Nguồn dữ liệu**: `/metrics` (`quality_avg`) / `data/logs.jsonl` (event `response_sent`)
- **Trường dữ liệu**: `quality_score`
- **Phép tổng hợp (Aggregation)**: `mean(quality_score)`
- **Đơn vị (Unit)**: `score_0_to_1` (thang điểm từ 0.00 đến 1.00)
- **Ngưỡng SLO / Threshold**: **Quality Avg $\ge$ 0.75**
- **Dạng biểu đồ**: Gauge Chart / Gauge Bar (Hiển thị điểm chất lượng trung bình từ 0.00 đến 1.00 so với mốc 0.75).

---

## 3. Mẫu dữ liệu JSON Endpoint `/metrics`

Khi gọi `GET http://localhost:8000/metrics`, hệ thống trả về snapshot dữ liệu có cấu trúc như sau:

```json
{
    "traffic": 25,
    "latency_p50": 1250.0,
    "latency_p95": 2800.0,
    "latency_p99": 3100.0,
    "avg_cost_usd": 0.0012,
    "total_cost_usd": 0.03,
    "tokens_in_total": 12500,
    "tokens_out_total": 4500,
    "error_rate_pct": 0.0,
    "error_breakdown": {},
    "quality_avg": 0.88
}
```

---

## 4. Kiểm tra hợp lệ Contract (Validator Status)

Chạy lệnh kiểm tra tính tương thích của Dashboard Contract:

```bash
python scripts/validate_dashboard.py
```

**Kết quả kiểm tra**:
> `HỢP LỆ: 6/6 panel có trong dashboard contract.`
