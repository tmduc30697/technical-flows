# Sequence - Base: Retry job sau backoff (không tách thời gian chờ)

Đây là flow **base**: job transcode 1080p thất bại do worker hết tài nguyên, hệ thống tự động retry sau một khoảng backoff. Toàn bộ thời gian chờ backoff bị tính gộp luôn vào duration của span retry, khiến "thời gian xử lý trung bình" của bước này bị tính sai lệch (bị đội lên bởi thời gian chờ, không phải thời gian xử lý thật). Đây là tiền đề cho yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    participant Queue as Job Queue
    participant T3 as Worker Transcode 1080p
    participant Tracer as Tracing Backend

    Queue->>T3: Chạy job transcode 1080p (lần 1)
    Note over T3: Trace #4, span bắt đầu
    T3-->>Queue: Lỗi, hết tài nguyên worker
    Note over T3: Span kết thúc với status=failed, duration chỉ tính phần xử lý
    Queue->>Queue: Chờ backoff 30s trước khi retry
    Queue->>T3: Chạy lại job transcode 1080p (lần 2)
    Note over T3: Span retry mới bắt đầu, nhưng start_time được tính từ lúc job được enqueue lại, gộp cả 30s chờ backoff vào duration
    T3-->>Queue: Xử lý thành công
    Note over T3: Span kết thúc, duration = 30s chờ + thời gian xử lý thật, không tách được phần nào là chờ
    T3->>Tracer: Gửi span retry với duration bị đội lên bởi thời gian backoff
```
