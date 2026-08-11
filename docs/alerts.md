# Alert và Runbook

Mọi alert dưới đây là symptom-based: điều kiện kích hoạt chỉ dùng chỉ số người dùng cảm nhận được (độ trễ, lỗi, chi phí), không dùng tên hàm/module nội bộ. Định nghĩa máy đọc được nằm ở `config/alert_rules.yaml`, ngưỡng SLO ở `config/slo.yaml`.

## Alert 1

- **Tên:** `high_latency_p95`
- **Severity:** warning
- **SLI/SLO liên quan:** `latency_p95_ms` — objective 3000 ms, target 99.5%
- **Điều kiện và thời gian duy trì:** `latency_p95 > 3000ms` duy trì liên tục **5 phút**
- **Ảnh hưởng tới người dùng:** Câu trả lời mất hơn 3 giây mới hiện. Người dùng cảm thấy ứng dụng treo, bấm gửi lại nhiều lần, làm tải tăng thêm. Chưa mất chức năng nhưng trải nghiệm xuống rõ rệt.
- **Ba bước kiểm tra đầu tiên:**
  1. Xác nhận triệu chứng còn sống và xem lệch giữa các phân vị — nếu P95 cao mà P50 vẫn bình thường thì chỉ một nhóm request bị chậm:
     ```bash
     curl -s http://localhost:8000/metrics | python -m json.tool
     ```
     Đọc `latency_p50` / `latency_p95` / `latency_p99`, đối chiếu panel **Latency P50/P95/P99** (time range 60 phút) xem thời điểm đường P95 vượt line 3000 ms.
  2. Kiểm tra incident flag có đang bật không — `rag_slow` chèn thẳng 2.5s vào mỗi request và là nguyên nhân hay gặp nhất:
     ```bash
     curl -s http://localhost:8000/health | python -m json.tool
     ```
     Xem khối `incidents`. Nếu `rag_slow: true` → đã tìm ra nguồn, sang phần mitigation.
  3. Lấy correlation ID của các request chậm nhất rồi mở trace tương ứng để xem span nào ăn thời gian:
     ```bash
     python -c "import json;rs=[json.loads(l) for l in open('data/logs.jsonl',encoding='utf-8') if l.strip()];rs=[r for r in rs if r.get('event')=='response_sent'];rs.sort(key=lambda r:r.get('latency_ms',0),reverse=True);[print(r['latency_ms'],r.get('correlation_id')) for r in rs[:5]]"
     ```
     Mở Langfuse → Traces, lọc theo `correlation_id` vừa lấy, xem waterfall: span `retrieve` dài bất thường là RAG, span `generate` dài là LLM.
- **Mitigation tạm thời:** Nếu incident flag đang bật thì tắt ngay:
  ```bash
  curl -X POST http://localhost:8000/incidents/rag_slow/disable
  ```
  Nếu không phải flag, giảm tải bằng cách hạ concurrency của client gọi vào và thông báo degraded mode cho người dùng trong khi điều tra tiếp.
- **Owner:** on-call-engineer

## Alert 2

- **Tên:** `elevated_error_rate`
- **Severity:** critical
- **SLI/SLO liên quan:** `error_rate_pct` — objective 2%, target 99.0%. Ngưỡng alert đặt ở 5%, cao hơn SLO để chỉ nổ khi thực sự nghiêm trọng.
- **Điều kiện và thời gian duy trì:** `error_rate_pct > 5` duy trì liên tục **3 phút**
- **Ảnh hưởng tới người dùng:** Cứ 20 câu hỏi thì hơn 1 câu trả về lỗi 500, không có câu trả lời nào. Đây là mất chức năng trực tiếp, nên severity critical và cửa sổ duy trì ngắn hơn Alert 1.
- **Ba bước kiểm tra đầu tiên:**
  1. Xác định lỗi thuộc loại nào — `error_breakdown` cho biết ngay là timeout, lỗi upstream hay lỗi validate:
     ```bash
     curl -s http://localhost:8000/metrics | python -m json.tool
     ```
     Đọc `error_rate_pct` và `error_breakdown`, đối chiếu panel **Error Rate & Breakdown** (refresh 15s).
  2. Kiểm tra incident flag `tool_fail` — flag này làm vector store ném `RuntimeError` ngay lập tức:
     ```bash
     curl -s http://localhost:8000/health | python -m json.tool
     ```
     Nếu `tool_fail: true` → đã tìm ra nguồn.
  3. Đọc log của chính các request hỏng, lấy correlation ID để mở trace:
     ```bash
     grep '"event": "request_failed"' data/logs.jsonl | tail -5 | python -m json.tool --json-lines
     ```
     Lấy `correlation_id` + `error_type`, mở Langfuse → Traces lọc theo ID đó, xem span nào ném lỗi. Kiểm tra thêm `prompt_source` trong metadata: nếu là `local-fallback` thì Langfuse đang không lấy được prompt.
- **Mitigation tạm thời:** Tắt flag nếu đang bật:
  ```bash
  curl -X POST http://localhost:8000/incidents/tool_fail/disable
  ```
  Nếu lỗi đến từ dependency ngoài, chuyển sang đường fallback (trả lời không cần RAG) và thông báo giảm chất lượng, hơn là để người dùng nhận 500.
- **Owner:** on-call-engineer

## Alert 3

- **Tên:** `cost_budget_exceeded`
- **Severity:** warning
- **SLI/SLO liên quan:** `daily_cost_usd` — objective 2.5 USD/ngày, target 100.0% (không cho phép vi phạm)
- **Điều kiện và thời gian duy trì:** `daily_cost_usd > 2.5`, kích hoạt ngay khi vượt, không cần cửa sổ duy trì vì đây là ngưỡng tích lũy đã vượt là vượt
- **Ảnh hưởng tới người dùng:** Chưa ảnh hưởng ngay tới chất lượng trả lời, nhưng khi hết ngân sách thì dịch vụ sẽ bị chặn hoặc phải hạ cấp model — lúc đó người dùng mất dịch vụ. Cảnh báo sớm để xử lý trước khi tới mức đó.
- **Ba bước kiểm tra đầu tiên:**
  1. Tách biệt "nhiều request" với "mỗi request đắt hơn" — đây là câu hỏi đầu tiên phải trả lời:
     ```bash
     curl -s http://localhost:8000/metrics | python -m json.tool
     ```
     So `traffic` với `avg_cost_usd`. Baseline là ~0.0022 USD/request. Nếu `traffic` tăng mà `avg_cost_usd` giữ nguyên → do lưu lượng. Nếu `avg_cost_usd` tăng vọt → do mỗi request phình ra.
  2. Nếu là request phình ra, xem tỉ lệ token trên panel **Token Usage**:
     ```bash
     curl -s http://localhost:8000/metrics | python -c "import json,sys;m=json.load(sys.stdin);print('ratio out/in =', round(m['tokens_out_total']/max(m['tokens_in_total'],1), 2))"
     ```
     Baseline ~4.2. Tỉ lệ nhảy lên gần 17 là dấu hiệu `cost_spike` (output nhân 4). Kiểm tra `curl -s http://localhost:8000/health` xem flag `cost_spike`.
  3. Nếu token bình thường mà chi phí vẫn cao, kiểm tra prompt version — prompt mới dài hơn sẽ đội input token:
     Mở Langfuse → Traces, xem metadata `prompt_name` / `prompt_label` / `prompt_version`, so cost trung bình giữa version cũ và mới. Đối chiếu `docs/PROMPT_VERSIONING.md`.
- **Mitigation tạm thời:** Tắt flag nếu đang bật:
  ```bash
  curl -X POST http://localhost:8000/incidents/cost_spike/disable
  ```
  Nếu do prompt mới, rollback label `production` về version trước. Nếu do lưu lượng thật, đặt rate limit tạm thời và xin nâng ngân sách.
- **Owner:** team-lead
