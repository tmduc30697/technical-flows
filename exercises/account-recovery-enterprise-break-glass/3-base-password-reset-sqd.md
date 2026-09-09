# Base sequence — Password reset (ad-hoc, chưa có quy trình chuẩn)

Đây là **base**, flow "Helpdesk reset mật khẩu cho nhân viên" ở trạng thái hiện tại — không có ticket, không có xác minh danh tính chuẩn hoá, không MFA, không log. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm siết lại đúng flow này.

```mermaid
sequenceDiagram
    actor Employee
    actor Agent as Helpdesk Agent
    participant DB as EMPLOYEE store

    Employee->>Agent: Báo quên mật khẩu (gọi điện/email trực tiếp)
    Agent->>Agent: Xác minh danh tính không theo chuẩn cố định (tuỳ agent)
    Agent->>DB: Đặt password_hash mới trực tiếp cho employee
    DB-->>Agent: Cập nhật thành công
    Agent-->>Employee: Đọc/gửi mật khẩu mới cho nhân viên
    Note over DB: Không ghi nhận agent nào đã reset, không bắt buộc đổi mật khẩu ở lần đăng nhập kế tiếp
```
