# Enhance sequence — Force-logout thủ công giải phóng slot gần như tức thời

Đây là **enhance** của flow `manage-devices` đã có ở base. So với base (chỉ cập nhật DB rồi để thiết bị bị logout tự phát hiện ở lần gọi API kế tiếp), enhance dùng lại cơ chế push realtime đã xây cho việc kick tự động (xem `6-enhance-kick-device-sqd.md`) để báo ngay cho thiết bị bị chọn. Đáp ứng yêu cầu 5: force-logout phải giải phóng slot gần như ngay lập tức, không để thiết bị khác phải chờ hoặc thử lại nhiều lần.

```mermaid
sequenceDiagram
    actor User
    participant App as Streaming Service
    participant DB as Database
    participant Push as Realtime Push Channel
    actor OldDevice as Thiết bị bị chọn logout
    actor NewDevice as Thiết bị khác đang chờ

    User->>App: Mở màn hình "Thiết bị đang đăng nhập"
    App->>DB: SELECT DEVICE_SESSION WHERE user_id=... AND status=active
    DB-->>App: Danh sách device session
    App-->>User: Hiển thị danh sách thiết bị

    User->>App: Chọn force-logout thiết bị X

    App->>DB: UPDATE DEVICE_SESSION SET status=kicked WHERE id=X
    App->>DB: INSERT DEVICE_KICK_EVENT (device_session_id=X, kicked_by=user, reason=manual_force_logout)

    par Báo ngay cho thiết bị bị chọn
        App->>Push: Publish kick-event tới thiết bị X (reason=manual_force_logout)
        Push-->>OldDevice: Nhận tín hiệu ngay lập tức
        OldDevice->>OldDevice: Dừng phát video ngay, hiển thị "Bạn đã bị đăng xuất từ thiết bị khác"
    end

    App-->>User: Xác nhận đã force-logout, slot đã được giải phóng

    NewDevice->>App: Bấm Play ngay sau đó
    App->>DB: SELECT COUNT(*) DEVICE_SESSION WHERE status=active
    DB-->>App: Slot của X đã giải phóng, còn chỗ trống
    App-->>NewDevice: Cho phép phát video ngay, không cần chờ hay retry
```
