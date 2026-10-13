# Checklist hoàn thành Lab 25B

Nguồn tổng hợp: `README.md`, `docs/RUBRIC.md`, `reports/REPORT_TEMPLATE.md` và các `TODO(student)` hiện có trong repo.

## 0. Chuẩn bị

- [ ] Xác nhận Python >= 3.10.
- [ ] Tạo/khởi động virtual environment `.venv`.
- [ ] Cài dependencies: `pip install -e ".[dev]"`.
- [ ] Chạy baseline và lưu kết quả ban đầu: `make baseline` hoặc `python scripts/run_gameday.py --client naive`.
- [ ] Chạy toàn bộ test một lần để ghi nhận trạng thái starter: `python -m pytest -q`.

## 1. Part A — Phân loại lỗi và retry (12 điểm)

File cần sửa: `src/gameday_lab/errors.py`, `src/gameday_lab/retry.py`.

- [ ] Cài đặt `classify()` cho đầy đủ các nhóm lỗi: rate limit, quota, overload, server, client và timeout.
- [ ] Cài đặt `is_retryable()`; tuyệt đối không retry lỗi 400 hoặc hết quota.
- [ ] Cài đặt `counts_toward_breaker()`; không để lỗi phía client làm trip breaker.
- [ ] Cài đặt `failover_helps()`; malformed request không được failover.
- [ ] Cài đặt full-jitter exponential backoff trong `FullJitterBackoff.delay()`.
- [ ] Bảo đảm delay không bao giờ nhỏ hơn `retry_after` khi provider trả giá trị này.
- [ ] Cài đặt token-bucket `RetryBudget.record_request()` và `try_acquire_retry()`.
- [ ] Dùng thời gian ảo `now`; không dùng `time.time()` hoặc `time.sleep()`.
- [ ] Chạy: `python -m pytest -q tests/test_errors_retry.py`.

## 2. Part B — Circuit breaker (12 điểm)

File cần sửa: `src/gameday_lab/breaker.py`.

- [ ] Cài đặt sliding window và chỉ đánh giá sau `min_calls`.
- [ ] Trip breaker theo tỷ lệ lỗi `failure_rate_threshold`.
- [ ] Tính successful-but-slow call vào tỷ lệ chậm và trip theo `slow_rate_threshold`.
- [ ] Cài đặt chuyển trạng thái CLOSED → OPEN → HALF_OPEN → CLOSED/OPEN.
- [ ] Giới hạn số probe đang bay bằng `half_open_max_probes`.
- [ ] Đóng breaker sau đủ `half_open_successes`; mở lại nếu probe lỗi hoặc chậm.
- [ ] Dùng `Permit.generation` để bỏ qua kết quả cũ sau khi breaker đổi thế hệ.
- [ ] Cài đặt `release()` và bảo đảm `probes_in_flight` không âm.
- [ ] Chạy: `python -m pytest -q tests/test_breaker.py`.

## 3. Part C — ResilientClient (30 điểm)

File cần sửa: `src/gameday_lab/client.py` và khối `client:` trong `configs/gameday.yaml`.

- [ ] Cài đặt `ladder(task)` theo thứ tự fallback và capability.
- [ ] Không bao giờ route tool call sang `gamma-mini`.
- [ ] Cài đặt `_route()` để bỏ qua provider có breaker không cấp permit.
- [ ] Cài đặt `start()` và nạp retry budget đúng một lần cho logical call.
- [ ] Cài đặt `on_success()` để ghi kết quả và latency vào đúng breaker/permit.
- [ ] Cài đặt decision table trong `on_error()`: classify → breaker → retry/budget/backoff → failover/give-up.
- [ ] Tôn trọng deadline tổng 30 giây và `retry_after`.
- [ ] Cài đặt `on_cancel()` để trả permit của attempt bị hủy.
- [ ] Bảo đảm chỉ có một tầng retry và không hard-code tên/thời điểm băng.
- [ ] Chạy: `python -m pytest -q tests/test_client.py`.
- [ ] Chạy `make compare`; tinh chỉnh cấu hình cho đến khi G1–G6 đều PASS.
- [ ] Kiểm tra riêng các điều kiện dễ mất điểm: G1 không vi phạm retry-after; G2 recovery <= 30s; G3 retry ratio <= 0,10; G4 p95 <= 10s; G5 không retry 400; G6 không incompatible route.

## 4. Part G — Semantic cache (10 điểm)

Làm ngay sau Part C. File cần sửa: `src/gameday_lab/cache.py` và khối `cache:` trong `configs/gameday.yaml`.

- [ ] Cài đặt cosine lookup trong đúng scope `prompt_version`.
- [ ] Không cache request cá nhân, kể cả cache theo user.
- [ ] Phân biệt fresh, stale còn dùng được và expired.
- [ ] Cài đặt TTL, `max_stale_s`, giới hạn `max_entries` và LRU eviction.
- [ ] Cài đặt `store()` và `invalidate(source)`.
- [ ] Cài đặt `CachingClient.start()` để trả fresh hit bằng `Serve`.
- [ ] Sau provider success, lưu `call.answer` hợp lệ vào cache.
- [ ] Khi inner client `GiveUp`, phục vụ stale nếu còn trong cửa sổ stale-if-error.
- [ ] Cài đặt webhook `on_content_update(source)` để invalidate đúng entry.
- [ ] Chạy: `python -m pytest -q tests/test_cache.py`.
- [ ] Chạy `make cache`; chỉnh threshold/TTL/max-stale đến khi C1 PASS.
- [ ] Xác nhận: `wrong_answers = 0`, `outdated_answers <= 15`, `faq_availability >= 93%`, `cache_hit_ratio >= 50%`.

## 5. Part D — Workflow exactly-once (12 điểm)

File cần sửa: `src/gameday_lab/workflows/executor.py`.

- [ ] Tạo idempotency key ổn định theo `(session, step)`; không dùng UUID ngẫu nhiên.
- [ ] Tạo checkpoint key ổn định cho từng bước.
- [ ] Cài đặt retry cho `ServiceError`/`ServiceTimeout` theo giới hạn cấu hình.
- [ ] Không bắt hoặc nuốt `WorkerCrash`.
- [ ] Resume bằng cách đọc checkpoint và chỉ chạy các bước chưa hoàn tất.
- [ ] Ghi checkpoint sau khi step thành công, không ghi trước side effect.
- [ ] Truyền idempotency key vào các bước có side effect để tránh refund/email trùng.
- [ ] Chạy: `python -m pytest -q tests/test_executor.py`.
- [ ] Chạy `make workflows`; xác nhận 0 duplicate refund, 0 duplicate email, 0 lost refund, completion >= 99%, wasted LLM/crash <= 0,6.

## 6. Part E — Burn-rate alerting và error budget (12 điểm)

File cần sửa: `src/gameday_lab/slo/alerting.py`.

- [ ] Cài đặt prefix sum trong `WindowSums` và truy vấn tổng theo cửa sổ.
- [ ] Cài đặt `burn_rate()` với error budget của SLO 99,5%.
- [ ] Cài đặt multi-window alerts: 14,4x (1h + 5m), 6x (6h + 30m), 1x (3d + 6h).
- [ ] Chỉ tạo một `AlertEvent` cho mỗi episode liên tục của từng rule.
- [ ] Cài đặt `error_budget_report()` gồm budget tiêu/còn lại và quyết định release freeze.
- [ ] Chạy: `python -m pytest -q tests/test_alerting.py`.
- [ ] Chạy `make slo`; kiểm tra page đúng sự cố lớn, ticket cho slow leak và không false-page cho blip.

## 7. Kiểm thử và chấm tự động

- [ ] Chạy toàn bộ unit tests: `python -m pytest -q`.
- [ ] Chạy `make gameday` và xác nhận 6/6 băng PASS.
- [ ] Chạy lại `make workflows`, `make slo`, `make cache`.
- [ ] Chạy trên nhiều seed nếu script hỗ trợ để tránh overfit dữ liệu public.
- [ ] Chạy `make grade` và kiểm tra `reports/scoreboard.json` (tối đa 88 điểm tự động).
- [ ] Kiểm tra repo trên môi trường sạch bằng luồng `pip install -e ".[dev]" && make grade`.

## 8. Part F — Report và postmortem (12 điểm)

- [ ] Copy `reports/REPORT_TEMPLATE.md` thành `reports/REPORT.md`.
- [ ] Điền họ tên/MSV, commit hash và điểm tự động.
- [ ] Điền bảng naive vs resilient cho đủ G1–G6 bằng số liệu thật.
- [ ] Giải thích từng tham số client bằng số liệu của băng liên quan.
- [ ] Thêm decision table/sơ đồ `on_error`.
- [ ] Thực hiện và ghi số cho 3 thí nghiệm: retry budget, timeout 30s, bỏ lọc tool calling.
- [ ] Điền kết quả workflow và giải thích vì sao cần cả idempotency key lẫn checkpoint.
- [ ] Điền kết quả alerting/error budget và quyết định release freeze.
- [ ] Chạy `make plot` để tạo hình G2 và dùng CSV timeseries làm bằng chứng.
- [ ] Viết postmortem G2 blameless: ít nhất 5 mốc có số, phân biệt trigger/root cause, giải thích metastable failure.
- [ ] Viết ít nhất 3 action item có loại, owner và tiêu chí đo được.
- [ ] Nêu một điểm yếu còn lại của client và hướng khắc phục.
- [ ] Điền mục cache: bảng kết quả, lý do tham số và 3 thí nghiệm threshold/TTL/stale-if-error.
- [ ] Trả lời cách alert khi cache giữ dashboard xanh nhưng provider đang sập.

## 9. Kiểm tra quy tắc và nộp bài

- [ ] Không sửa `src/gameday_lab/sim/`, `naive_client.py`, workflow services/runner/naive executor, SLO data/scoring, scenario YAML hoặc `tests/`.
- [ ] Không hard-code thời điểm sự cố, tên băng hoặc tên tài liệu.
- [ ] Kiểm tra `reports/` có `REPORT.md`, hình/CSV cần thiết và `scoreboard.json`.
- [ ] Đặt repo public đúng mẫu: `Track3-Day25-<MSV>-<Ho-Va-Ten>`.
- [ ] Commit toàn bộ code, config và report lên nhánh `main`.
- [ ] Cập nhật commit hash cuối cùng trong `reports/REPORT.md` (nếu chỉnh report sau đó, ghi rõ hash bài nộp phù hợp quy trình lớp).
- [ ] Push GitHub và mở link repo ở chế độ ẩn danh để xác nhận public.
- [ ] Nộp link repo lên kênh của lớp.

## Thứ tự ưu tiên đề xuất

1. Part A → Part B → Part C (đường găng của toàn bài).
2. Part G ngay sau C vì băng cache phụ thuộc `ResilientClient`.
3. Part D và E độc lập, có thể làm sau core client.
4. Chạy toàn bộ benchmark/grade để sinh số liệu.
5. Viết report từ artifact đã sinh, rồi kiểm tra và nộp bài.
