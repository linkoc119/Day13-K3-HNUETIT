# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm:
- Repository URL:
- Commit SHA cuối:
- Thành viên và vai trò:
  - **Thành viên A (API & Middleware)**: CP1 Middleware, gán Correlation ID, bổ sung exception handler.
  - **Thành viên B (Security Engineer)**: CP1 PII Scrubbing, regex patterns & kiểm chứng log che PII.
  - **Thành viên C (Metrics & Dashboard)**: CP1/CP2 đo đếm error_rate_pct & thiết kế spec Dashboard 6 nhóm chỉ số.
  - **Thành viên D (SRE & Alerts Engineer)**: CP2 Thiết lập SLO, viết Alert rules & Alert Runbook xử lý sự cố.
  - **Thành viên E (QA & Chief Investigator)**: Chạy load test, bọc trace cho sub-component RAG/LLM, dẫn dắt điều tra Challenge (CP3) & hoàn thiện báo cáo.

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: **30/100** *(Baseline CP0)*
- Tổng số traces:
- Số PII leak còn lại:
- Link/đường dẫn dashboard:

## 3. Logging và tracing

- Evidence correlation ID:
- Evidence PII redaction:
- Evidence trace waterfall:
- Giải thích một span đáng chú ý:

## 4. Prompt versioning

- Prompt name:
- Version/label baseline:
- Version/label candidate:
- Trace ID của mỗi version:
- Bằng chứng đổi label hoặc rollback:

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`:
- Evidence dashboard:
- SLO đã chọn và lý do:
- Alert rules và runbook:

## 6. Điều tra challenge

- Challenge ID:
- Triệu chứng từ metrics:
- Trace ID liên quan:
- Log line/correlation ID liên quan:
- Root cause:
- Fix action:
- Preventive measure:

## 7. Đóng góp cá nhân

Với mỗi thành viên, ghi rõ nhiệm vụ và link commit/PR tương ứng.

| Thành viên | Vai trò | Phần việc chính | Commit/PR | Điều đã học |
|---|---|---|---|---|
| **Thành viên A** | API & Middleware | CP1 Middleware, gán Correlation ID, bổ sung exception handler | | |
| **Thành viên B** | Security Engineer | CP1 PII Scrubbing, regex patterns và kiểm chứng log không lộ PII | | |
| **Thành viên C** | Metrics & Dashboard | CP1/CP2 đo đếm error_rate_pct và thiết kế spec Dashboard 6 nhóm chỉ số | | |
| **Thành viên D** | SRE & Alerts Engineer | CP2 Thiết lập SLO, viết Alerts rules và Alert Runbook xử lý sự cố | | |
| **Thành viên E** | QA & Chief Investigator | Chạy load test, bọc trace sub-component RAG/LLM, dẫn dắt điều tra Challenge (CP3) & hoàn thiện báo cáo nhóm | | |
