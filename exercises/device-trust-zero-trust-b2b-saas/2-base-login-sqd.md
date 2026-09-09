# Base sequence — Login chỉ xác thực mật khẩu, không quan tâm thiết bị

Đây là **base**, flow đăng nhập hiện tại: server chỉ kiểm tra email/mật khẩu đúng là cấp session ngay, không hỏi gì về thiết bị đang dùng để truy cập. Đây là tiền đề để đề bài "có nghĩa" — chính vì base không phân biệt được thiết bị công ty quản lý với thiết bị cá nhân nên mới cần bổ sung device posture verification.

```mermaid
sequenceDiagram
    actor User
    participant Laptop as Thiết bị bất kỳ (công ty hoặc cá nhân)
    participant App as Internal Tool
    participant DB as Database

    User->>Laptop: Nhập email + mật khẩu
    Laptop->>App: POST /login (email, password)
    App->>DB: Kiểm tra password_hash khớp
    DB-->>App: Hợp lệ

    App->>DB: INSERT SESSION (user_id, status=active)
    DB-->>App: Session tạo thành công

    App-->>Laptop: Trả session token
    Note over App,Laptop: Không có bước nào kiểm tra thiết bị này có được công ty quản lý hay không
    Laptop-->>User: Truy cập dữ liệu tài chính, nhân sự nhạy cảm thành công
```
