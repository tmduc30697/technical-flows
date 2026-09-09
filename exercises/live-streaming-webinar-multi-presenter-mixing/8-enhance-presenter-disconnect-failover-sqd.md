# Enhance sequence — Presenter disconnect failover

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có cơ chế phát hiện/khắc phục khi presenter đang live bị rớt kết nối. Đáp ứng **yêu cầu 3** (tự động phát hiện và chuyển output sang nguồn còn sống trong thời gian ngắn nhất khi presenter chính đang phát bị rớt kết nối).

```mermaid
sequenceDiagram
    actor PresenterA as Presenter A (đang là nguồn chính)
    participant IngestA as Ingest Presenter A
    participant Monitor as Connection Monitor
    participant Mixer
    actor Viewer

    Mixer->>Viewer: Đang phát output từ Presenter A
    Note over PresenterA,IngestA: Presenter A rớt kết nối đột ngột giữa buổi
    Monitor->>IngestA: Kiểm tra heartbeat/frame liên tục
    IngestA--xMonitor: Không có frame mới trong khoảng thời gian ngắn
    Monitor->>Monitor: Xác nhận DISCONNECT_EVENT(presenter=A, detected_at=now)
    Monitor->>Mixer: Yêu cầu chuyển nguồn khẩn cấp
    Mixer->>Mixer: Chọn fallback_target (slide cuối cùng của A, hoặc presenter khác đang online)
    Mixer-->>Viewer: Chuyển sang fallback trong thời gian ngắn nhất, tránh đứng hình kéo dài
    PresenterA->>IngestA: Kết nối lại sau đó
    IngestA->>Monitor: Heartbeat trở lại bình thường
    Monitor->>Monitor: Ghi recovered_at=now vào DISCONNECT_EVENT
    Monitor-->>Mixer: Presenter A có thể được chuyển trở lại nếu host chọn
```
