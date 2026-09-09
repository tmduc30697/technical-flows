# Base sequence — Xem danh sách thiết bị và force-logout thủ công

Đây là **base**, flow người dùng vào màn hình "Thiết bị đang đăng nhập" để xem danh sách `DEVICE_SESSION` đang active và chủ động logout một thiết bị. Đây là tiền đề cho yêu cầu 5 của đề bài (force-logout phải giải phóng slot gần như ngay lập tức) — ở base, việc release slot chỉ cập nhật DB, thiết bị bị logout không nhận được tín hiệu chủ động nào để dừng phát ngay mà phải tự phát hiện qua lần gọi API tiếp theo.

```mermaid
sequenceDiagram
    actor User
    participant App as Streaming Service
    participant DB as Database
    participant OldDevice as Thiết bị bị logout

    User->>App: Mở màn hình "Thiết bị đang đăng nhập"
    App->>DB: SELECT DEVICE_SESSION WHERE user_id=... AND status=active
    DB-->>App: Danh sách device session
    App-->>User: Hiển thị danh sách thiết bị

    User->>App: Chọn logout thiết bị X
    App->>DB: UPDATE DEVICE_SESSION SET status=ended WHERE id=X
    DB-->>App: Cập nhật thành công
    App-->>User: Xác nhận đã logout thiết bị X

    Note over App,OldDevice: Không có kênh nào chủ động báo cho thiết bị X biết nó đã bị logout
    OldDevice->>App: (một lúc sau) Gọi API tiếp theo, mới nhận lỗi session invalid
    App-->>OldDevice: Lỗi session invalid, không rõ nguyên nhân, video vẫn phát dở dang trước đó
```
