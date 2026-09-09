# Enhance sequence — Change sensitive info (second factor + cảnh báo thiết bị tin cậy)

Đây là **enhance**, cùng flow "Change sensitive info" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 3 của đề bài: OTP không còn là factor duy nhất/đủ mạnh — cần thêm 1 factor độc lập (xác nhận từ thiết bị tin cậy), và nếu còn thiết bị tin cậy đang đăng nhập thì phải cảnh báo tới đó trước khi thực thi, cho chủ tài khoản thật cơ hội chặn kịp thời.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant DB as SENSITIVE_ACTION_REQUEST / SIM_SWAP_EVENT store
    actor TrustedDevice as Thiết bị tin cậy khác (nếu có)

    User->>App: Yêu cầu đổi email liên kết / phương thức khôi phục (qua OTP)
    App->>DB: Xác minh OTP (như base) → phone_otp_status=verified
    App->>DB: Kiểm tra có SIM_SWAP_EVENT đang cooling_off_until > now cho user không
    alt Đang trong cooling-off
        DB-->>App: Có sự kiện SIM-swap gần đây, còn cooling-off
        App-->>User: Từ chối tạm thời, yêu cầu chờ hết cooling-off hoặc dùng kênh xác minh khác
    else Không trong cooling-off
        App->>DB: Tạo SENSITIVE_ACTION_REQUEST (status=pending)
        App->>DB: Kiểm tra có DEVICE trusted=true đang có SESSION active không
        alt Có thiết bị tin cậy đang đăng nhập
            App->>TrustedDevice: Gửi TRUSTED_DEVICE_ALERT (nội dung yêu cầu thay đổi vừa phát sinh)
            alt Chủ tài khoản thật xác nhận/approve trên thiết bị tin cậy
                TrustedDevice-->>App: response=approved (chính là second factor độc lập)
                App->>DB: second_factor_status=verified
                App->>DB: Áp dụng thay đổi, status=completed
                App-->>User: Đổi thành công
            else Chủ tài khoản thật bấm "Không phải tôi" / block
                TrustedDevice-->>App: response=blocked
                App->>DB: status=blocked, tạo/nâng risk cho SIM_SWAP_EVENT
                App-->>User: Yêu cầu bị chặn, tài khoản được bảo vệ
            else Không phản hồi trong thời gian chờ
                TrustedDevice-->>App: response=no_response (timeout)
                App->>DB: status=blocked (an toàn theo hướng từ chối)
                App-->>User: Yêu cầu bị treo, cần xác minh qua kênh khác
            end
        else Không có thiết bị tin cậy nào đang đăng nhập
            App-->>User: Yêu cầu thêm factor độc lập khác (vd mã xác nhận qua backup_email)
        end
    end
```
