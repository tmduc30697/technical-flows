# Sequence Diagram — Base: End Meeting and Process Recording

Đây là **base**, flow xử lý recording khi cuộc họp kết thúc theo cách thông thường (hàng đợi chuẩn, không ưu tiên, không cô lập đặc biệt, không tách nguồn) — tiền đề bắt buộc mà enhance sẽ tăng tốc và bổ sung thêm khả năng.

```mermaid
sequenceDiagram
    participant MeetingSvc as Meeting Service
    participant Storage as Recording Storage
    participant Queue as Standard Transcode Queue
    participant Worker as Transcode Worker

    MeetingSvc->>MeetingSvc: Meeting ended
    MeetingSvc->>Storage: Store RECORDING (raw file)
    MeetingSvc->>Queue: Enqueue transcode job (cùng độ ưu tiên với mọi video khác)
    Queue->>Worker: Dispatch khi tới lượt (có thể chờ lâu nếu hàng đợi bận)
    Worker->>Storage: Transcode ra bản ghi xem được
    Worker-->>MeetingSvc: Recording ready
    Note over MeetingSvc,Queue: Không có SLA thời gian, không cô lập tài nguyên theo từng khách hàng
```
