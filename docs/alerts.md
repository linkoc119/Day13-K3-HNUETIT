# Alert và Runbook

Mỗi alert dưới đây dựa trên triệu chứng người dùng nhìn thấy hoặc SLO, không dựa vào tên implementation nội bộ.

## Alert 1

- Tên: high_latency_p95
- Severity: warning
- SLI/SLO liên quan: p95 latency của request `/chat`
- Điều kiện và thời gian duy trì: p95 latency lớn hơn 3000 ms trong 5 phút
- Ảnh hưởng tới người dùng: câu trả lời đến chậm, trải nghiệm hỏi đáp bị gián đoạn
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency và xác nhận spike theo time range 5-10 phút gần nhất.
  2. Lọc trace có latency cao nhất, xem span RAG/LLM/tool nào chiếm thời gian.
  3. Dùng correlation ID từ trace để tìm log `request_received` và `response_sent`.
- Mitigation tạm thời: giảm concurrency, rollback prompt/version gây tăng token, hoặc tắt incident/practice flag nếu đang bật.
- Owner: observability

## Alert 2

- Tên: elevated_error_rate
- Severity: critical
- SLI/SLO liên quan: error rate của API
- Điều kiện và thời gian duy trì: request_failed / request_received lớn hơn 2% trong 5 phút
- Ảnh hưởng tới người dùng: request lỗi 5xx, agent không trả lời được
- Ba bước kiểm tra đầu tiên:
  1. Mở panel errors để xem error_type nổi bật.
  2. Lấy correlation ID của log `request_failed` mới nhất.
  3. Kiểm tra trace tương ứng để biết lỗi nằm ở retrieval, prompt fetch hay LLM generation.
- Mitigation tạm thời: rollback phiên bản prompt, tắt dependency không ổn định, hoặc chuyển sang local prompt fallback.
- Owner: api

## Alert 3

- Tên: quality_score_drop
- Severity: warning
- SLI/SLO liên quan: quality proxy score
- Điều kiện và thời gian duy trì: mean quality_score nhỏ hơn 0.75 trong 10 phút
- Ảnh hưởng tới người dùng: câu trả lời kém liên quan dù hệ thống vẫn trả 200
- Ba bước kiểm tra đầu tiên:
  1. Mở panel quality và so sánh trước/sau thay đổi prompt label.
  2. Lọc trace theo `prompt_name`, `prompt_label`, `prompt_version`.
  3. Đọc log answer preview để xác nhận câu trả lời thiếu context hay bị redaction quá mức.
- Mitigation tạm thời: rollback prompt label về production ổn định, tăng kiểm tra retrieval, hoặc giảm phạm vi feature bị ảnh hưởng.
- Owner: ai-quality
