# Sequence Diagram - Enhance: queue-wait-timeout

Đây là flow **enhance mới**: khi lock đang bị pipeline khác giữ, pipeline mới phải xếp hàng chờ với `max_wait_seconds` xác định, nếu chờ vượt quá giới hạn thì job bị huỷ, có thông báo cho người trigger, và có thể được cấu hình để tự động retry sau đó. Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    actor Dev as Developer / CI trigger
    participant CDC as Pipeline Region C
    participant Coord as Lock Coordinator ensemble (ZK/etcd)
    participant Queue as Deploy Queue
    participant Audit as Audit Log
    participant Alert as Alerting/Ops

    Dev->>CDC: Trigger deploy Region C
    CDC->>Coord: Request acquire lock "global-deploy-lock"
    Coord-->>CDC: Lock đang bị region khác giữ
    CDC->>Queue: Enqueue, max_wait_seconds = 300
    loop Chờ tới lượt hoặc hết thời gian
        Queue->>Coord: Kiểm tra lock đã rảnh chưa
        Coord-->>Queue: Vẫn đang bị giữ
    end
    alt Hết max_wait_seconds trước khi tới lượt
        Queue->>Audit: Ghi log QUEUE_TIMEOUT (region=C, pipeline chờ quá hạn)
        Queue->>Alert: Gửi cảnh báo cho người vận hành, job bị huỷ
        Queue-->>CDC: Huỷ job chờ (status=CANCELLED)
        CDC-->>Dev: Thông báo deploy Region C bị huỷ do chờ lock quá lâu, có thể retry thủ công/tự động
    else Lock được nhả trước khi hết thời gian chờ
        Coord-->>Queue: Lock rảnh, cấp cho pipeline đầu hàng đợi
        Queue-->>CDC: Cấp lock, tiếp tục deploy Region C bình thường
    end
```
