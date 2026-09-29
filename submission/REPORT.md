# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Báo cáo dùng đường dẫn tương đối tới evidence. Output text hiện có nằm trong `submission/evidence/`; ảnh Langfuse/dashboard cần bổ sung sau khi chụp từ giao diện.

## 1. Thông tin học viên

- **Họ và tên:** Nguyen Ngoc Vinh
- **MSSV:** 2A202602833
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/VinhNN747/K4-L3-DAY13-NguyenNgocVinh-2A202602833-Monitoring-LLMOps.git
- **Commit SHA cuối:** Chưa commit
- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** day13-k4-l3a-2A202602833

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | evidence/01-pytest.txt |
| Log validator | evidence/02-log-validator.txt |
| Dashboard validator | evidence/03-dashboard-validator.txt |
| Structured log | evidence/04-structured-log.txt |
| PII redaction | evidence/05-pii-redaction.txt |
| Trace list | evidence/06-trace-list.txt |
| Trace waterfall | evidence/07-trace-waterfall.txt |
| Trace metadata | evidence/08-trace-metadata.txt |
| Prompt versions | evidence/09-prompt-versions.txt |
| Prompt rollback | evidence/10-prompt-rollback.txt |
| Dashboard runtime | evidence/11-dashboard-overview.txt |
| Incident metric | evidence/12-incident-metric.txt |
| Incident log | evidence/13-incident-log.txt |
| Incident trace | evidence/14-incident-trace.txt |
| Challenge run | evidence/15-challenge-run.txt |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| validate_logs.py | 30/100 | 100/100 | 93 records, 43 correlation IDs, 0 PII leak |
| validate_dashboard.py | 6/6 | 6/6 | Đủ sáu panel |
| pytest | 22 passed | 23 passed | Thêm test response `x-request-id` và `x-response-time-ms` |
| Số traces hợp lệ | 0 | 12 | 36 observations, có root/retriever/generation |
| Số PII leak | chưa đo | 0 | Recursive scrubber |
| Latency P95 / TTFT P95 | — | 6505 ms / 50 ms | CP3 challenge `rag_slow` |
| Retrieval success rate | — | 100% | 5/5 retrieval thành công |

## 4. Logging và PII

- Cách tạo/nhận và truyền correlation ID: middleware nhận request header hợp lệ hoặc sinh req-xxxxxxxx, bind vào contextvars và trả lại qua response header.
- Các metadata được ghi vào structured log: correlation_id, user_id_hash, session_id, feature, model, env, latency_ms, ttft_ms, tool_name và tool_success.
- Cách bảo đảm PII được scrub trước khi ghi: scrubber đệ quy qua dict/list/tuple trước JSONL writer; detector gồm email, phone Việt Nam, CCCD và credit card.
- Cách kiểm chứng kết quả: validate_logs.py đạt 100/100 và PII leak bằng 0; test middleware xác nhận response giữ `x-request-id` hợp lệ và trả `x-response-time-ms`.

## 5. Tracing và prompt versioning

- Cách xác nhận traces do chính tôi tạo trong project cá nhân: query Langfuse trong time window workload và đối chiếu correlation ID với `data/logs.jsonl`; có 12 traces/36 observations.
- Cấu trúc root/retrieval/generation observations: root `lab-agent-run` có child `retrieval` dạng RETRIEVER và `fake-llm` dạng GENERATION; evidence `07-trace-waterfall.txt`.
- Cách nối trace với log: correlation ID nằm trong metadata Langfuse và structured log; ví dụ `req-29babfaa` trong `08-trace-metadata.txt`.
- Prompt name: day13-chat.
- Version/label baseline: v1 / baseline, trace 89beff6b714b2a119cff911ea7fde91d.
- Version/label candidate: v2 / candidate, trace 5dc6d5adbd2083c6123ed6361e9fd30b.
- Trace ID của mỗi version: baseline `89beff6b714b2a119cff911ea7fde91d`; candidate `5dc6d5adbd2083c6123ed6361e9fd30b`; production sau promote `02d6f7504c23112931a53b4a1df040a1`; production sau rollback `2b50855f61c47d879ffca464b5e9bcb1`.
- Cách promote và rollback production: promote v2 rồi chạy trace 02d6...; rollback v1 rồi chạy trace 2b508...; trạng thái cuối xác nhận production=1.

## 6. Dashboard, SLO và alerts

- Dashboard và sáu panel: latency, traffic, errors, cost, tokens, quality; time range 60 phút, unit và threshold hiển thị.
- SLO và lý do chọn: successful requests trong 28 ngày, target 99.5%, nhằm giữ latency <= 3000 ms và theo dõi độ tin cậy.
- Cách tính error budget: 100% - 99.5% = 0.5%.
- Ba alert và runbook tương ứng: high_latency_p95 → latency runbook; high_error_rate → API/error runbook; low_retrieval_success → retrieval runbook trong docs/alerts.md.

## 7. Điều tra challenge

- Challenge ID: day13-k4-l3a-monitoring-llmops-v1.
- Khoảng thời gian điều tra: 2026-09-29T15:34:23Z – 2026-09-29T15:34:50Z.
- Triệu chứng từ metrics: 5 requests, latency P95/P99 4708 ms, TTFT P95 51 ms, vượt challenge threshold 2000 ms; retrieval success 100%.
- Log line và correlation ID liên quan: response_sent với `req-983b8514`, `req-a5f65be0`, `req-2e2573b7`, `req-5768ed3a`, `req-096aec2a`.
- Trace ID và span gây ảnh hưởng: trace `9ad2e5d5b80f776bc6a3e4a7314725fc`, retrieval span `e05c9da770cbf587`, được ghi trong `14-incident-trace.txt`.
- Root cause: incident chính thức rag_slow thêm delay vào retrieval.
- Fix action: chạy scripts/inject_incident.py --disable; kết quả disable xác nhận các incident đều false.
- Preventive measure: giữ alert P95 3000 ms, SLO/error budget và quy trình Metrics → Logs → Traces; tiếp tục theo dõi retrieval success.

## 8. Giải thích và tự đánh giá

- Một quyết định kỹ thuật quan trọng và lý do: dùng observation adapter no-op để app vẫn chạy khi Langfuse disabled/test double, nhưng khi enabled vẫn có nested retrieval/generation trace.
- Một lỗi/blocker đã gặp: lần đọc đầu tiên dùng Langfuse observations API với field mặc định nên không thấy model/usage/cost/metadata; một lần fetch prompt đầu tiên timeout và app dùng fallback minh bạch.
- Cách tìm nguyên nhân và xử lý: kiểm tra observation fields `core,basic,metadata,model,usage,prompt,metrics`, đối chiếu `prompt_source`, sau đó warm cache và xác nhận trace candidate dùng managed prompt.
- Cách hiểu luồng Metrics → Logs → Traces: metrics cho biết triệu chứng và ngưỡng; log cung cấp correlation ID; trace phân rã thời gian theo span để định vị retrieval.
- Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM: version/label giúp truy xuất thay đổi, token/cost giúp kiểm soát chi phí, SLO định nghĩa mức chấp nhận được, rollback đưa production về version ổn định.
- Điều quan trọng nhất đã học: cần nối ba nguồn bằng correlation ID và kiểm tra đúng field selection của API.
- Hạn chế hoặc phần chưa hoàn thành: cần tạo commit cuối và cập nhật SHA; challenge trace/prompt evidence đã có trong thư mục evidence.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [x] Các output text hiện có mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
