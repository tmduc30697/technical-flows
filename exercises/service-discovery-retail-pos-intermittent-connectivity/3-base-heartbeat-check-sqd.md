# Sequence Diagram — Base: Heartbeat Check

Đây là **base**, flow heartbeat nhị phân hiện tại — thiết bị im lặng quá ngưỡng timeout cố định thì bị báo động ngay, không phân biệt được cửa hàng đóng cửa bình thường hay thiết bị hỏng thật. Đây chính là điểm sẽ bị thay đổi ở enhance.

```mermaid
sequenceDiagram
    participant Device as Thiết bị POS
    participant Platform as Nền tảng quản lý POS
    actor Ops as Đội vận hành

    loop mỗi chu kỳ cố định
        Device->>Platform: Heartbeat
        Platform->>Platform: Update last_heartbeat_at
    end

    Device--xPlatform: Không gửi heartbeat (quá timeout cố định)
    Platform->>Platform: Mark DEVICE status=offline
    Platform->>Ops: Cảnh báo thiết bị offline
    Note over Platform,Ops: Không phân biệt được đây là cửa hàng đóng cửa hay thiết bị hỏng thật
```
