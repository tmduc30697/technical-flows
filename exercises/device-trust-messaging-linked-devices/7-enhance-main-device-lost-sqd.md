# Enhance sequence — Chính sách khi thiết bị chính bị mất/đăng xuất

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Khi thiết bị chính (điện thoại) đăng xuất hoặc bị đánh dấu mất, các thiết bị liên kết đang hoạt động không bị thu hồi ngay lập tức mà có 1 khoảng grace period để người dùng liên kết lại với thiết bị chính mới, sau đó tự động hết hạn nếu không có hành động gì. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Phone as Điện thoại (thiết bị chính)
    participant Server
    participant DB as Database
    participant Web as Web (thiết bị liên kết đang hoạt động)
    participant Job as Grace Period Expiry Job

    User->>Phone: Đăng xuất khỏi điện thoại (hoặc báo mất máy qua kênh hỗ trợ)
    Phone->>Server: Yêu cầu đăng xuất/đánh dấu mất thiết bị chính
    Server->>DB: UPDATE DEVICE_SESSION(Phone) SET status=revoked
    Server->>DB: INSERT MAIN_DEVICE_TRANSITION (transition_type=logged_out, old_main_session_id=Phone, grace_period_ends_at=now+7 ngày)

    Note over Web,Server: Web vẫn tiếp tục hoạt động bình thường trong grace period, không bị đăng xuất ngay

    alt Người dùng liên kết lại với thiết bị chính mới trong grace period
        User->>Server: Cài app trên điện thoại mới, đăng nhập lại bằng OTP
        Server->>DB: INSERT DEVICE_SESSION mới (role=main), UPDATE MAIN_DEVICE_TRANSITION SET new_main_session_id=điện thoại mới, transition_type=relinked
        Server-->>Web: Thông báo đã có thiết bị chính mới, Web tiếp tục hoạt động bình thường, gắn với thiết bị chính mới
    else Hết grace period mà chưa có thiết bị chính mới nào
        Job->>DB: SELECT MAIN_DEVICE_TRANSITION WHERE grace_period_ends_at <= now() AND new_main_session_id IS NULL
        DB-->>Job: Danh sách user chưa liên kết lại thiết bị chính
        Job->>DB: UPDATE DEVICE_SESSION SET status=expired WHERE role=linked AND user_id thuộc danh sách trên
        Job-->>Web: Đẩy thông báo buộc đăng xuất do hết hạn, yêu cầu liên kết lại từ đầu khi có thiết bị chính mới
    end
```
