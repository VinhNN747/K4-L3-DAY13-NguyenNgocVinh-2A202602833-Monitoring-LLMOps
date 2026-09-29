# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: `high_latency_slo_breach`
- Severity: Critical
- Duration: 10 phút liên tục
- Kênh thông báo: Slack
- SLI/SLO liên quan: fast_successful_requests, latency P95 ≤ 3000 ms
- Điều kiện và thời gian duy trì: `latency_p95_ms > 3000` trong 10 phút
- Ảnh hưởng tới người dùng: phản hồi chậm, timeout tăng
- Ba bước kiểm tra đầu tiên: xem latency/TTFT; lọc `response_sent` theo thời gian; mở trace có P95 cao và so sánh retrieval/generation
- Mitigation tạm thời: giảm tải/concurrency, tắt feature gây chậm nếu cần
- Owner: platform-oncall

## Alert 2

- Tên: `elevated_error_rate`
- Severity: Critical
- Duration: 5 phút liên tục
- Kênh thông báo: Slack
- SLI/SLO liên quan: successful requests, error rate ≤ 2%
- Điều kiện và thời gian duy trì: `error_rate_pct > 2` trong 5 phút
- Ảnh hưởng tới người dùng: request thất bại hoặc trả lời không đầy đủ
- Ba bước kiểm tra đầu tiên: xem error breakdown; lấy correlation ID từ log; mở trace và kiểm tra span lỗi
- Mitigation tạm thời: rollback prompt/feature thay đổi gần nhất, bật fallback
- Owner: api-oncall

## Alert 3

- Tên: `retrieval_success_degraded`
- Severity: Warning
- Duration: 10 phút liên tục
- Kênh thông báo: Slack
- SLI/SLO liên quan: retrieval success rate ≥ 90%
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90` trong 10 phút
- Ảnh hưởng tới người dùng: câu trả lời thiếu ngữ cảnh hoặc kém chính xác
- Ba bước kiểm tra đầu tiên: kiểm tra tool success; phân nhóm theo feature; mở retriever span để xác định timeout/rỗng kết quả
- Mitigation tạm thời: dùng cached context/fallback và giảm timeout retry storm
- Owner: rag-oncall
