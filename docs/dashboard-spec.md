# Yêu cầu dashboard

Contract kiểm tra bằng máy: `config/dashboard.yaml`. Hướng dẫn dựng và kiểm tra runtime: [DASHBOARD_SETUP.md](DASHBOARD_SETUP.md).

Nguồn dữ liệu của toàn bộ panel: endpoint `GET /metrics` (`app/metrics.py::snapshot`).

Tiêu chuẩn chung áp dụng cho mọi panel:

- Time range mặc định: **60 phút**.
- Auto refresh: **15–30 giây**.
- Mọi panel phải hiện đơn vị và threshold/SLO line.
- Lớp chính giữ đúng 6 panel dưới đây.

Giá trị ví dụ lấy từ một lần chạy thật (`python scripts/load_test.py`, 10 request):

```bash
curl http://localhost:8000/metrics | python -m json.tool
```

```json
{
    "traffic": 10,
    "latency_p50": 1091.0,
    "latency_p95": 1516.0,
    "latency_p99": 1516.0,
    "avg_cost_usd": 0.0022,
    "total_cost_usd": 0.0216,
    "tokens_in_total": 330,
    "tokens_out_total": 1377,
    "error_breakdown": {},
    "error_rate_pct": 0.0,
    "quality_avg": 0.88
}
```

## 1. Latency

- **Tên panel:** Latency P50/P95/P99
- **Field từ `/metrics`:** `latency_p50`, `latency_p95`, `latency_p99`
- **Đơn vị:** milliseconds (ms)
- **Time range mặc định:** 60 phút, refresh 20s
- **Threshold/SLO line:** đường ngang đỏ tại **3000 ms** trên series P95 (SLO `latency_p95_ms`, target 99.5%); đường vàng cảnh báo tại 2400 ms (80% ngân sách)
- **Công cụ:** Grafana time series (3 series) — Langfuse Latency chart dùng để đối chiếu
- **Giá trị ví dụ:** P50 1091 ms · P95 1516 ms · P99 1516 ms → đang ở ~51% ngân sách SLO

## 2. Traffic

- **Tên panel:** Request Volume
- **Field từ `/metrics`:** `traffic`
- **Đơn vị:** requests (counter tích lũy); hiển thị kèm dạng dẫn xuất req/min
- **Time range mặc định:** 60 phút, refresh 20s
- **Threshold/SLO line:** không phải SLO. Đường tham chiếu baseline **10 req/run**; đánh dấu vùng bất thường khi traffic = 0 quá 10 phút (mất traffic) hoặc tăng > 3× baseline
- **Công cụ:** Grafana bar chart / stat panel — Langfuse Traces count để đối chiếu
- **Giá trị ví dụ:** 10 requests

## 3. Error

- **Tên panel:** Error Rate & Breakdown
- **Field từ `/metrics`:** `error_rate_pct` (series chính), `error_breakdown` (bảng phụ theo `error_type`)
- **Đơn vị:** phần trăm (%) cho error rate; count cho breakdown
- **Time range mặc định:** 60 phút, refresh 15s (refresh nhanh nhất vì đây là tín hiệu sự cố)
- **Threshold/SLO line:** đường vàng tại **2%** (SLO `error_rate_pct`, target 99.0%), đường đỏ tại **5%** (ngưỡng alert `elevated_error_rate`)
- **Công cụ:** Grafana time series + table — nguồn phụ: `data/logs.jsonl` lọc `event=request_failed`
- **Giá trị ví dụ:** 0.0% · breakdown rỗng `{}`

## 4. Cost

- **Tên panel:** Cost over Time
- **Field từ `/metrics`:** `total_cost_usd` (tích lũy), `avg_cost_usd` (mỗi request)
- **Đơn vị:** USD
- **Time range mặc định:** 60 phút cho lớp chính; panel phụ 24 giờ để so với ngân sách ngày
- **Threshold/SLO line:** đường đỏ tại **2.5 USD/ngày** (SLO `daily_cost_usd`); đường tham chiếu **0.005 USD/request** cho `avg_cost_usd` — vượt ngưỡng này là dấu hiệu cost spike
- **Công cụ:** Grafana time series (total) + stat (avg) — Langfuse Cost dashboard để đối chiếu theo model
- **Giá trị ví dụ:** total 0.0216 USD · avg 0.0022 USD/request → 0.86% ngân sách ngày

## 5. Tokens

- **Tên panel:** Token Usage (Input/Output)
- **Field từ `/metrics`:** `tokens_in_total`, `tokens_out_total`
- **Đơn vị:** tokens
- **Time range mặc định:** 60 phút, refresh 30s
- **Threshold/SLO line:** không phải SLO. Đường tham chiếu tỉ lệ **output/input ≈ 4.2** từ baseline; đánh dấu bất thường khi tỉ lệ > 8 (dấu hiệu `cost_spike`, output phình gấp ~4 lần)
- **Công cụ:** Grafana stacked bar (in vs out) — Langfuse Generations token usage để đối chiếu
- **Giá trị ví dụ:** in 330 · out 1377 → tỉ lệ 4.17

## 6. Quality

- **Tên panel:** Quality Score (proxy)
- **Field từ `/metrics`:** `quality_avg`
- **Đơn vị:** điểm 0.0–1.0
- **Time range mặc định:** 60 phút, refresh 30s
- **Threshold/SLO line:** đường đỏ tại **0.75** (SLO `quality_score_avg`, target 95.0%). Đây là ngưỡng sàn — cảnh báo khi giá trị **giảm xuống dưới** đường, ngược chiều với các panel khác
- **Công cụ:** Grafana time series với ngưỡng dưới — Langfuse Scores để đối chiếu theo prompt version
- **Giá trị ví dụ:** 0.88 → trên sàn SLO 0.13 điểm

## Kiểm tra contract trước khi chụp evidence

```bash
python scripts/validate_dashboard.py
```

Screenshot phải nhìn rõ tên panel, đơn vị và khoảng thời gian.
