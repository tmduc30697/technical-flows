# Sequence - Enhance: Retry job sau backoff (tách riêng thời gian chờ)

Đây là flow **enhance** của `job-retry-backoff` (so với base). Khác biệt so với base: span retry ghi rõ `attempt_number=2` và có field `backoff_wait_ms` riêng biệt lưu thời gian cố ý delay, tách hoàn toàn khỏi `duration_ms` là thời gian xử lý thật, để số liệu "thời gian xử lý trung bình" của bước transcode 1080p không bị tính sai bởi thời gian chờ backoff. Đáp ứng **yêu cầu 3** của đề bài.

```mermaid
sequenceDiagram
    participant Queue as Job Queue
    participant T3 as Worker Transcode 1080p
    participant Tracer as Tracing Backend

    Queue->>T3: Chạy job transcode 1080p (attempt_number=1)
    Note over T3: Span con trong trace gốc, job_type=transcode, attempt_number=1
    T3-->>Queue: Lỗi, hết tài nguyên worker
    Note over T3: Span kết thúc, status=failed, duration_ms chỉ tính phần xử lý thật
    Queue->>Queue: Chờ backoff 30s trước khi retry
    Note over Queue: Khoảng chờ này được ghi nhận riêng, chưa gắn vào span xử lý nào
    Queue->>T3: Chạy lại job transcode 1080p (attempt_number=2)
    Note over T3: Span mới, attempt_number=2, backoff_wait_ms=30000 ghi riêng, duration_ms chỉ tính thời gian xử lý thật của lần retry này
    T3-->>Queue: Xử lý thành công
    Note over T3: Span kết thúc, status=success
    T3->>Tracer: Gửi span attempt=2 với backoff_wait_ms và duration_ms tách bạch rõ ràng
    Note over Tracer: Khi tính latency trung bình bước transcode, chỉ cộng duration_ms, bỏ qua backoff_wait_ms
```
