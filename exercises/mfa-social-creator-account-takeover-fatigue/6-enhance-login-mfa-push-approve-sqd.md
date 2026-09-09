# Sequence Diagram — Enhance: Login với MFA Push Approve (chống fatigue)

Đây là **enhance** của flow `login-mfa-push-approve` ở base. So với base, flow này thay đổi ở 3 điểm: (1) mỗi push đều mang đủ ngữ cảnh (vị trí, thiết bị, thời gian) kèm number matching thay vì nút Approve đơn giản, (2) số push gửi liên tiếp trong khoảng thời gian ngắn bị giới hạn, request mới sau ngưỡng sẽ tự động bị khoá, (3) khi phát hiện một chuỗi từ chối liên tiếp — dấu hiệu tấn công — hệ thống tự động khoá đăng nhập từ nguồn phát sinh và cảnh báo chủ tài khoản qua kênh độc lập, không chờ họ tự nhận ra.

```mermaid
sequenceDiagram
    actor Attacker
    actor Owner as Creator (chủ tài khoản)
    participant Auth as Auth Service
    participant Limiter as Push Rate Limiter
    participant Push as Push Notification Service
    participant Mobile as Authenticator App (Mobile)
    participant Monitor as Security Monitor
    participant Alert as Alert Channel (email hoặc SMS dự phòng)

    Attacker->>Auth: POST /login (password đã lộ, hợp lệ)

    loop Attacker spam login liên tục
        Auth->>Limiter: Kiểm tra PUSH_RATE_LIMIT_WINDOW cho user
        alt Chưa vượt ngưỡng trong window
            Limiter-->>Auth: Cho phép, tăng push_count
            Auth->>Auth: Tạo MFA_CHALLENGE (device_info, location, ip, display_code)
            Auth->>Push: Gửi push kèm đầy đủ ngữ cảnh + display_code
            Push->>Mobile: Hiển thị vị trí, thiết bị, thời gian, yêu cầu nhập display_code
            Mobile-->>Owner: Thấy notification lạ, không khớp thao tác của mình
            Owner->>Mobile: Từ chối (Deny), không nhập code
            Mobile->>Auth: Challenge DENIED
            Auth->>Auth: Ghi nhận denial, tăng consecutive_denial_count
        else Đã vượt ngưỡng push trong window
            Limiter-->>Auth: locked_until chưa hết hạn, từ chối tạo challenge mới
            Auth-->>Attacker: Chặn tạm thời, không gửi thêm push
        end
    end

    Auth->>Monitor: consecutive_denial_count vượt ngưỡng
    Monitor->>Auth: Xác nhận dấu hiệu tấn công, tạo SECURITY_LOCK_EVENT
    Auth->>Auth: Khoá đăng nhập từ source_ip phát sinh request
    Auth->>Alert: Gửi SECURITY_ALERT qua kênh độc lập (không phải kênh push đang bị tấn công)
    Alert-->>Owner: "Chúng tôi đã chặn các yêu cầu đăng nhập đáng ngờ vào tài khoản của bạn"
    Auth-->>Attacker: Đăng nhập bị khoá tạm thời từ nguồn này
```

**So với base:** thêm bước kiểm tra `Push Rate Limiter` trước mỗi challenge, thêm ngữ cảnh + number matching trong nội dung push, và thêm nhánh phát hiện tấn công (`Security Monitor`) tự động khoá + cảnh báo độc lập — không còn chỉ là "gửi push, chờ 1 cú chạm Approve" như base.
