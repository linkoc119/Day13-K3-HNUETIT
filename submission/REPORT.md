# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm: Day13-K3-HNUETIT
- Repository URL: https://github.com/linkoc119/Day13-K3-HNUETIT
- Commit SHA cuối: cd84f4f
- Thành viên và vai trò:
  - Ngô Hùng Phúc - 2A202601069: Logging & PII, tổng hợp report/evidence.
  - Nguyễn Văn Linh - 2A202601971: Metrics, SLO và kiểm tra log validation.
  - Nguyễn Duy Hoàng - 2A202601147: Tracing và prompt versioning.
  - Nguyễn Ngọc Dương - 2A202601717: Dashboard contract và alert rules.
  - Lê Văn Long - 2A20261711: Incident investigation và runbook.

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: 100/100
- Tổng số traces: 10 traces trên Langfuse Cloud sau khi chạy `python scripts/load_test.py`.
- Số PII leak còn lại: 0
- Link/đường dẫn dashboard: `config/dashboard.yaml`, kiểm tra bằng `python scripts/validate_dashboard.py`

## 3. Logging và tracing

- Evidence correlation ID: `submission/evidence/technical-evidence.md`
- Evidence PII redaction: `data/logs.jsonl` có `[REDACTED_EMAIL]`, `[REDACTED_PHONE_VN]`, `[REDACTED_CREDIT_CARD]`
- Evidence trace waterfall: `submission/evidence/traces_detail.png`
- Giải thích một span đáng chú ý: span generation của `LabAgent.run` gắn metadata prompt, model, token, cost và query preview để nối metric latency/cost sang trace cụ thể.

## 4. Prompt versioning

- Prompt name: `day13-chat`
- Version/label baseline: `production`
- Version/label candidate: `candidate`
- Trace ID của mỗi version: xem danh sách traces trong `submission/evidence/traces.png` và metadata prompt trong `submission/evidence/traces_prompt.png`.
- Bằng chứng đổi label hoặc rollback: code hỗ trợ đổi label bằng `LANGFUSE_PROMPT_LABEL`; evidence prompt/metadata nằm trong `submission/evidence/traces_prompt.png`.

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`: `HỢP LỆ: 6/6 panel có trong dashboard contract.`
- Evidence dashboard: `config/dashboard.yaml`, `submission/evidence/technical-evidence.md`, `submission/evidence/traces.png`
- SLO đã chọn và lý do: p95 latency <= 3000 ms, error rate <= 2%, quality score mean >= 0.75 để bao phủ tốc độ, độ ổn định và chất lượng AI.
- Alert rules và runbook: `config/alert_rules.yaml`, `docs/alerts.md`

## 6. Điều tra challenge

- Challenge ID: chưa có release chính thức từ Lab Coach trong repo hiện tại.
- Triệu chứng từ metrics: practice hiện có thể dùng `rag_slow`, `tool_fail` qua `scripts/inject_incident.py`.
- Trace ID liên quan: xem trace list/detail trong `submission/evidence/traces.png` và `submission/evidence/traces_detail.png`.
- Log line/correlation ID liên quan: có thể dùng correlation ID trong `data/logs.jsonl`, ví dụ `req-9dc84710`.
- Root cause: với practice `rag_slow`, root cause dự kiến nằm ở span retrieval/RAG latency tăng.
- Fix action: rollback prompt/incident flag, kiểm tra retrieval dependency, giảm payload hoặc cache kết quả truy xuất.
- Preventive measure: alert p95 latency, trace metadata prompt/version, và runbook kiểm tra metrics -> traces -> logs.

## 7. Đóng góp cá nhân

| Thành viên | Phần việc | Commit/PR | Điều đã học |
|---|---|---|---|
| Ngô Hùng Phúc - 2A202601069 | Hoàn thiện logging context, PII redaction, tổng hợp report/evidence | Repo commit cuối sau khi push | Biết nối correlation ID, JSON logs, metrics và trace metadata để tìm root cause có bằng chứng |
| Nguyễn Văn Linh - 2A202601971 | Kiểm tra metrics, `validate_logs.py`, SLO latency/error/quality | Repo commit cuối sau khi push | Hiểu cách dùng log có cấu trúc để đo traffic, latency, token và cost |
| Nguyễn Duy Hoàng - 2A202601147 | Phân tích tracing, prompt name/label/version và fallback prompt | Repo commit cuối sau khi push | Hiểu cách trace gắn prompt version giúp rollback và so sánh chất lượng |
| Nguyễn Ngọc Dương - 2A202601717 | Kiểm tra dashboard 6 panel và alert rules | Repo commit cuối sau khi push | Biết thiết kế dashboard theo latency, traffic, errors, cost, tokens và quality |
| Lê Văn Long - 2A20261711 | Viết runbook và hướng điều tra incident metrics -> traces -> logs | Repo commit cuối sau khi push | Hiểu quy trình xác định triệu chứng, khoanh vùng span và chứng minh root cause bằng log |
