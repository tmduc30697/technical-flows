# Enhance sequence — Kick thiết bị cũ khi thiết bị mới giành chỗ lúc đã hết slot

Đây là **enhance**, flow hoàn toàn mới phát sinh khi tài khoản đã dùng hết `max_concurrent_devices` và có thêm một thiết bị mới muốn phát. Đáp ứng yêu cầu 2: thiết bị cũ (được chọn là thiết bị ít hoạt động nhất theo `last_heartbeat_at`) bị đá ra để nhường chỗ, phải dừng phát ngay và hiển thị rõ lý do, không để người dùng gặp lỗi timeout khó hiểu.

```mermaid
sequenceDiagram
    actor NewDevice as Thiết bị mới
    participant App as Streaming Service
    participant Lock as Redis (distributed lock)
    participant DB as Database
    participant Push as Realtime Push Channel
    actor OldDevice as Thiết bị cũ (ít hoạt động nhất)

    NewDevice->>App: Bấm Play
    App->>Lock: SET slot-lock:user123 NX PX 500ms
    Lock-->>App: Giành lock thành công

    App->>DB: SELECT COUNT(*) DEVICE_SESSION WHERE status=active
    DB-->>App: count = max, hết slot

    App->>DB: SELECT DEVICE_SESSION ORDER BY last_heartbeat_at ASC LIMIT 1
    DB-->>App: Trả về session của OldDevice, ít hoạt động nhất

    App->>DB: UPDATE DEVICE_SESSION SET status=kicked WHERE id=OldDevice
    App->>DB: INSERT DEVICE_KICK_EVENT (device_session_id=OldDevice, kicked_by=new_device, reason=slot_taken_by_new_device)
    App->>DB: INSERT DEVICE_SESSION (NewDevice, status=active, fencing_token=N+1)
    App->>Lock: DEL slot-lock:user123

    App-->>NewDevice: Cho phép phát video

    par Gửi tín hiệu kick thời gian thực
        App->>Push: Publish kick-event tới OldDevice (reason=slot_taken_by_new_device)
        Push-->>OldDevice: Nhận tín hiệu kick ngay lập tức
        OldDevice->>OldDevice: Dừng phát video ngay
        OldDevice-->>OldDevice: Hiển thị "Bạn đã bị đăng xuất do thiết bị khác đang xem, đã hết slot cho phép"
    end

    Note over OldDevice: Không phải chờ timeout mới biết lý do, tín hiệu kick đến gần như tức thời
```
