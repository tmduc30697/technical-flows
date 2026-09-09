# Base sequence — Acquire slot khi bấm play (race condition chưa xử lý)

Đây là **base**, flow chiếm slot hiện tại: server đếm số `DEVICE_SESSION` đang active rồi so sánh với `max_concurrent_devices` bằng một transaction đọc-rồi-ghi thông thường, không có khóa nào bảo vệ khoảng giữa lúc đếm và lúc ghi. Đây chính là kịch bản race condition nêu ở yêu cầu 1 của đề bài, khi hai thiết bị mới cùng bấm play gần như đồng thời lúc chỉ còn 1 slot trống.

```mermaid
sequenceDiagram
    actor DeviceA as Thiết bị A (mới)
    actor DeviceB as Thiết bị B (mới)
    participant App as Streaming Service
    participant DB as Database

    Note over App,DB: Giả sử user còn đúng 1 slot trống trên tổng max_concurrent_devices

    DeviceA->>App: Bấm Play
    DeviceB->>App: Bấm Play (gần như cùng lúc)

    App->>DB: (A) SELECT COUNT(*) DEVICE_SESSION WHERE status=active
    DB-->>App: (A) count = max-1, còn 1 slot

    App->>DB: (B) SELECT COUNT(*) DEVICE_SESSION WHERE status=active
    DB-->>App: (B) count = max-1, còn 1 slot

    Note over App: Cả 2 request đều đọc thấy còn slot trống, chưa có khóa chặn giữa đọc và ghi

    App->>DB: (A) INSERT DEVICE_SESSION status=active
    App->>DB: (B) INSERT DEVICE_SESSION status=active

    DB-->>App: Cả 2 INSERT đều thành công
    App-->>DeviceA: Cho phép phát video
    App-->>DeviceB: Cho phép phát video

    Note over DB: Vượt quá max_concurrent_devices, oversell slot do race condition
```
