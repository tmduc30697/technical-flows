# Sequence Diagram — Base: Mark Live Event

Đây là **base**, flow đánh dấu mốc thời gian quan trọng lúc đang live (donate, highlight) — tiền đề cho yêu cầu "VOD phải giữ đúng các mốc thời gian này sau khi ghép và transcode".

```mermaid
sequenceDiagram
    actor Viewer
    participant LiveSvc as Live Streaming Service
    participant EventStore as Event Store

    Viewer->>LiveSvc: Donate / trigger highlight action
    LiveSvc->>LiveSvc: Compute occurred_at (relative to stream start)
    LiveSvc->>EventStore: Create LIVE_EVENT (stream_id, type, occurred_at)
    EventStore-->>LiveSvc: Event recorded
    LiveSvc-->>Viewer: Acknowledge
```
