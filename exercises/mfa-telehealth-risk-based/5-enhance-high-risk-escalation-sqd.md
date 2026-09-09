# Enhance sequence — Chặn và xác minh mạnh hơn khi rủi ro cao bất thường

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm điểm rủi ro nên không thể phân biệt rủi ro cao bất thường). Đáp ứng yêu cầu: khi điểm rủi ro cao bất thường (ví dụ đăng nhập từ hai quốc gia cách nhau vài phút), phải chặn và yêu cầu xác minh bổ sung mạnh hơn thay vì chỉ hỏi lại OTP thông thường.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Auth Service
    participant Risk as Risk Scoring Service
    participant DB as LOGIN_SESSION store
    participant Esc as ESCALATED_VERIFICATION
    participant Log as MFA_DECISION_LOG
    actor Support as Bộ phận hỗ trợ

    User->>App: Nhập email + mật khẩu (kèm device fingerprint, IP)
    App->>Risk: Yêu cầu chấm điểm rủi ro
    Risk-->>App: RISK_ASSESSMENT(risk_level=critical, ví dụ đăng nhập 2 quốc gia cách nhau vài phút)

    App->>DB: Cập nhật LOGIN_SESSION(mfa_verified=false)
    App->>Log: Ghi quyết định mfa_required=true, reason="impossible_travel_critical_risk"
    App->>Esc: Tạo ESCALATED_VERIFICATION(method=contact_support, status=pending)
    Note over App: Không cho phép xác minh bằng OTP thông thường ở mức rủi ro này
    App-->>User: Chặn đăng nhập, yêu cầu liên hệ hỗ trợ để xác minh danh tính

    User->>Support: Liên hệ hỗ trợ, cung cấp thông tin xác minh bổ sung
    Support->>Support: Xác minh danh tính thủ công (giấy tờ, câu hỏi bảo mật...)
    alt Xác minh thành công
        Support->>Esc: Cập nhật ESCALATED_VERIFICATION(status=resolved)
        Support->>DB: Cho phép App mở lại phiên đăng nhập mới cho user
        App-->>User: Có thể đăng nhập lại bình thường
    else Xác minh thất bại hoặc nghi ngờ gian lận
        Support->>Esc: Cập nhật ESCALATED_VERIFICATION(status=resolved, kết quả=từ chối)
        Support-->>User: Từ chối, khóa tài khoản tạm thời, hướng dẫn quy trình khiếu nại
    end
```
