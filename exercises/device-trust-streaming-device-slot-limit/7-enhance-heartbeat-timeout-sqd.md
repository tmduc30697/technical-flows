# Enhance sequence — Heartbeat định kỳ và tự động giải phóng slot khi timeout

Đây là **enhance**, flow hoàn toàn mới bổ sung tín hiệu heartbeat từ client và job nền quét session hết hạn. Đáp ứng yêu cầu 3: xác định thiết bị đã ngừng hoạt động (app bị kill nền, mất mạng, đóng tab) qua ngưỡng timeout hợp lý, tránh vừa giữ slot ảo cho thiết bị đã tắt thật sự vừa tránh giải phóng nhầm slot của thiết bị chỉ mất kết nối tạm thời.

```mermaid
sequenceDiagram
    actor Device as Thiết bị đang phát
    participant App as Streaming Service
    participant DB as Database
    participant Sweeper as Background Sweeper Job

    loop Mỗi heartbeat_interval_sec (vd 30s)
        Device->>App: Heartbeat (device_session_id)
        App->>DB: UPDATE DEVICE_SESSION SET last_heartbeat_at=now() WHERE id=...
    end

    Note over Device,App: Thiết bị bị mất mạng đột ngột, app bị kill nền, không còn gửi heartbeat nữa

    loop Mỗi chu kỳ quét (vd 15s)
        Sweeper->>DB: SELECT DEVICE_SESSION WHERE status=active AND last_heartbeat_at < now() - heartbeat_timeout_sec
        DB-->>Sweeper: Trả về các session quá hạn heartbeat_timeout_sec (vd 90s)
    end

    alt Session vượt ngưỡng timeout thật sự
        Sweeper->>DB: UPDATE DEVICE_SESSION SET status=expired WHERE id=...
        Sweeper->>DB: INSERT DEVICE_KICK_EVENT (kicked_by=system, reason=heartbeat_timeout)
        Note over Sweeper,DB: Slot được giải phóng, thiết bị khác có thể chiếm chỗ
    else Thiết bị chỉ mất kết nối tạm thời và gửi lại heartbeat trước khi hết timeout
        Device->>App: Heartbeat gửi lại kịp trước ngưỡng timeout
        App->>DB: UPDATE last_heartbeat_at=now()
        Note over Sweeper,DB: Sweeper không đánh dấu expired, slot vẫn được giữ nguyên cho thiết bị này
    end
```
