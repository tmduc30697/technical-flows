# Sequence Diagram — Base: Live Segment Ingest

Đây là **base**, flow ghi hình trong lúc đang live — tiền đề bắt buộc cho hậu xử lý VOD: chính flow này tạo ra các `SEGMENT` (.ts nhỏ, có thể thiếu/lỗi do gián đoạn mạng) mà sau này phải được ghép lại đúng thứ tự thời gian.

```mermaid
sequenceDiagram
    actor Streamer
    participant Encoder as Live Encoder
    participant LiveSvc as Live Streaming Service
    participant Storage as Segment Storage

    Streamer->>Encoder: Start broadcasting
    loop mỗi vài giây trong lúc live
        Encoder->>LiveSvc: Push encoded segment (.ts, sequence_number)
        LiveSvc->>Storage: Store SEGMENT
        Storage-->>LiveSvc: Segment stored
        LiveSvc-->>Encoder: Ack
        Note over LiveSvc,Storage: Segment có thể bị thiếu hoặc lỗi nếu mạng gián đoạn giữa chừng
    end
    Streamer->>LiveSvc: End stream
    LiveSvc->>LiveSvc: Mark STREAM status = ended
```
