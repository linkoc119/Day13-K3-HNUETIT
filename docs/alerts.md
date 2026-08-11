# Quy trình xử lý cảnh báo (Alert Runbooks)

Tài liệu hướng dẫn xử lý sự cố (Runbook) cho 3 quy tắc cảnh báo symptom-based dựa trên SLO của hệ thống AI Observability.

---

## Alert 1: High Latency P95 (high_latency_p95)

- **Tên**: `high_latency_p95`
- **Severity**: `warning`
- **SLI/SLO liên quan**: `latency_p95_ms` (Objective: $\le$ 3000ms, Target: 99.5%)
- **Điều kiện và thời gian duy trì**: `latency_p95 > 3000ms for 5 minutes`
- **Ảnh hưởng tới người dùng**: Người dùng gặp tình trạng thời gian phản hồi câu trả lời lâu (> 3 giây), trải nghiệm bị chậm hoặc bị timeout ở phía client application.
- **Ba bước kiểm tra đầu tiên**:
  1. **Kiểm tra Dashboard Metrics**: Mở panel `Latency percentiles` xem P95/P99 tăng đột biến từ thời điểm nào và mức độ ảnh hưởng toàn hệ thống.
  2. **Mở Langfuse Traces**: Tìm các Trace ID có latency > 3000ms, xem sơ đồ Waterfall để xác định bước gây nghẽn (bước RAG `mock_rag` retriever hay bước LLM `mock_llm` generation).
  3. **Tra cứu Log Context**: Tìm log record trong `data/logs.jsonl` bằng `correlation_id` tương ứng với trace bị chậm để kiểm tra sự cố kịch bản (`rag_slow`), độ dài prompt input, hoặc lỗi nghẽn tài nguyên.
- **Mitigation tạm thời**:
  - Giảm tham số `top_k` của RAG retriever để rút ngắn thời gian tìm kiếm context.
  - Bật caching phản hồi cho các câu hỏi phổ biến hoặc chuyển hướng sang fallback model nhẹ hơn.
  - Tắt kịch bản incident nếu đang ở môi trường test: `python scripts/inject_incident.py --disable`.
- **Owner**: `on-call-engineer` (SRE / Member D)

---

## Alert 2: Elevated Error Rate (elevated_error_rate)

- **Tên**: `elevated_error_rate`
- **Severity**: `critical`
- **SLI/SLO liên quan**: `error_rate_pct` (Objective: $\le$ 2%, Target: 99.0%)
- **Điều kiện và thời gian duy trì**: `error_rate_pct > 5 for 3 minutes`
- **Ảnh hưởng tới người dùng**: Tỷ lệ lớn người dùng (trên 5%) không nhận được câu trả lời, nhận phản hồi lỗi HTTP 500/503 hoặc hệ thống gián đoạn dịch vụ.
- **Ba bước kiểm tra đầu tiên**:
  1. **Kiểm tra Panel Errors**: Mở panel `Error rate and breakdown` trên Dashboard để xác định nhóm lỗi chiếm đa số (`LLMError`, `RAGError`, `Timeout`, v.v.).
  2. **Trích xuất Stacktrace trong Log**: Lọc file log `data/logs.jsonl` theo `event == "request_failed"` hoặc `level == "ERROR"` để lấy nguyên nhân chi tiết và `correlation_id`.
  3. **Kiểm tra Endpoint `/health` & Provider**: Gọi `curl http://localhost:8000/health` và kiểm tra kết nối tới dịch vụ LLM/RAG bên thứ 3 (API Key, Quota status, Network connectivity).
- **Mitigation tạm thời**:
  - Bật cơ chế Fallback Mode: Trả về câu trả lời mặc định hoặc chuyển hướng request sang provider phụ.
  - Áp dụng Circuit Breaker tự động ngắt các request tới sub-component bị lỗi.
  - Khởi động lại API service nếu bị treo bộ nhớ hoặc deadlock.
- **Owner**: `on-call-engineer` (SRE / Member D)

---

## Alert 3: Cost Budget Exceeded (cost_budget_exceeded)

- **Tên**: `cost_budget_exceeded`
- **Severity**: `warning`
- **SLI/SLO liên quan**: `daily_cost_usd` (Objective: $\le$ $2.5/ngày, Target: 100.0%)
- **Điều kiện và thời gian duy trì**: `daily_cost_usd > 2.5`
- **Ảnh hưởng tới người dùng**: Không gây lỗi trực tiếp cho người dùng cuối nhưng nguy cơ làm cạn kiệt ngân sách vận hành và vỡ chỉ số chi phí tài chính của dự án.
- **Ba bước kiểm tra đầu tiên**:
  1. **Phân tích Panel Cost & Tokens**: Kiểm tra panel `Cost over time` và `Input and output tokens` trên Dashboard xem chi phí tăng vọt do Spike Traffic (nhiều request) hay do Prompt/Output quá dài.
  2. **Soi Chi Tiết Langfuse Traces**: Tìm các Trace ID có `cost_usd` và token count cao nhất để kiểm tra nội dung prompt và số lượng token nạp/ra.
  3. **Kiểm Tra Spam & Loop Log**: Rà soát `data/logs.jsonl` xem có `user_id_hash` hoặc `session_id` nào gửi request liên tục (retry loop hoặc nghi vấn tấn công DDoSH/spam prompt).
- **Mitigation tạm thời**:
  - Áp dụng Rate Limiting khắt khe hơn trên Middleware cho các session tiêu tốn nhiều tài nguyên.
  - Giới hạn `max_tokens` của LLM response và rút gọn bớt system prompt / RAG context.
- **Owner**: `team-lead` (Team Lead / SRE)
