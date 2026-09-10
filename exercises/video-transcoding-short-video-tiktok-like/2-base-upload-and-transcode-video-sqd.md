# Sequence Diagram — Base: Upload and Transcode Video

Đây là **base**, flow upload và transcode thông thường (không SLA riêng, không kiểm tra sơ bộ, không autoscale, không hủy job) — tiền đề bắt buộc mà enhance sẽ tối ưu độ trễ và bổ sung các cơ chế vận hành.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant VideoSvc as Video Service
    participant Queue as Transcode Queue (chung)
    participant Worker as Transcode Worker

    User->>App: Upload video
    App->>VideoSvc: Submit video (codec, aspect_ratio, framerate, duration gốc từ thiết bị)
    VideoSvc->>VideoSvc: Create VIDEO, status = uploaded
    VideoSvc->>Queue: Enqueue TRANSCODE_JOB (cùng độ ưu tiên với mọi video)
    Queue->>Worker: Dispatch khi tới lượt
    Worker->>Worker: Transcode ra 1 định dạng chuẩn
    Worker-->>VideoSvc: Job completed
    VideoSvc-->>User: Video xem được (thời gian chờ không cố định)
```
