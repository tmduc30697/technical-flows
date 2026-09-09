# Enhance sequence — Acquire slot bằng distributed lock, tránh race cho slot cuối

Đây là **enhance** của flow `acquire-slot` đã có ở base. So với base, thay vì đọc-đếm rồi ghi trực tiếp, server phải giành một distributed lock ngắn hạn theo `user_id` trên Redis trước khi đếm và ghi slot, cấp `fencing_token` tăng dần cho mỗi session mới. Đáp ứng yêu cầu 1: đảm bảo hai thiết bị mới cùng giành slot cuối không thể cùng vượt giới hạn, đồng thời lock chỉ bọc phần đếm+ghi rất ngắn (vài ms) nên không làm chậm đáng kể việc bắt đầu phát video.

```mermaid
sequenceDiagram
    actor DeviceA as Thiết bị A (mới)
    actor DeviceB as Thiết bị B (mới)
    participant App as Streaming Service
    participant Lock as Redis (distributed lock)
    participant DB as Database

    Note over App,Lock: Giả sử user còn đúng 1 slot trống trên tổng max_concurrent_devices

    DeviceA->>App: Bấm Play
    DeviceB->>App: Bấm Play (gần như cùng lúc)

    App->>Lock: (A) SET slot-lock:user123 NX PX 500ms
    Lock-->>App: (A) Giành lock thành công

    App->>Lock: (B) SET slot-lock:user123 NX PX 500ms
    Lock-->>App: (B) Thất bại, lock đang bị A giữ

    App->>DB: (A) SELECT COUNT(*) DEVICE_SESSION WHERE status=active
    DB-->>App: (A) count = max-1, còn 1 slot
    App->>DB: (A) INSERT DEVICE_SESSION status=active, fencing_token=N+1
    DB-->>App: (A) Ghi thành công
    App->>Lock: (A) DEL slot-lock:user123 (release ngay sau khi ghi xong)

    App-->>DeviceA: Cho phép phát video ngay (độ trễ thêm chỉ ~vài ms do lock)

    App->>Lock: (B) Retry SET slot-lock:user123 NX PX 500ms sau vài chục ms
    Lock-->>App: (B) Giành lock thành công (A đã release)
    App->>DB: (B) SELECT COUNT(*) DEVICE_SESSION WHERE status=active
    DB-->>App: (B) count = max, hết slot
    App->>Lock: (B) DEL slot-lock:user123
    App-->>DeviceB: Từ chối, hiển thị "Đã đạt giới hạn thiết bị xem đồng thời"
```
