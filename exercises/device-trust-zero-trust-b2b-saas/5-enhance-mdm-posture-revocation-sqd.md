# Enhance sequence — Buộc kết thúc session ngay khi MDM báo posture xấu đi

Đây là **enhance**, flow hoàn toàn mới xử lý sự kiện MDM báo cáo real-time trong lúc session đang hoạt động. Đáp ứng yêu cầu 3: khi thiết bị chuyển sang trạng thái không đạt chuẩn (tắt mã hóa ổ đĩa, jailbreak/root), hệ thống phải buộc kết thúc session hiện tại gần như ngay lập tức, không chờ session tự hết hạn theo `expires_at`.

```mermaid
sequenceDiagram
    actor User
    participant Device as Thiết bị đang có session active
    participant MDM as MDM Service
    participant App as Internal Tool
    participant DB as Database

    Note over User,App: User đang có session active, truy cập bình thường

    MDM->>MDM: Phát hiện thiết bị tắt mã hóa ổ đĩa (hoặc bị jailbreak/root)
    MDM->>App: Webhook posture-changed (device_id, event_type=posture_degraded, detail=disk_encryption_disabled)

    App->>DB: INSERT POSTURE_EVENT (device_id, event_type=posture_degraded, detail, reported_at)
    App->>DB: SELECT SESSION WHERE device_id=... AND status=active
    DB-->>App: Session đang active tìm thấy

    App->>DB: UPDATE SESSION SET status=force_terminated WHERE id=...
    App->>Device: Gửi tín hiệu invalidate session (push/websocket hoặc chặn ở lần request kế tiếp)

    Device->>App: Request tiếp theo bất kỳ (API call, refresh trang)
    App-->>Device: 401, "Session đã bị chấm dứt do thiết bị không còn đạt chuẩn bảo mật, vui lòng liên hệ IT"

    Note over App,DB: Thời gian từ lúc MDM báo cáo tới lúc session bị chấm dứt chỉ vài giây, không chờ tới expires_at
```
