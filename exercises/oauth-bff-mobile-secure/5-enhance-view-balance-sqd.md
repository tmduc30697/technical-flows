# Enhance sequence — View balance

Đây là **enhance**, flow "Xem số dư tài khoản". So với base, BFF không chỉ trả về số dư tài khoản chính mà còn lặp qua các `LINKED_BANK_CONNECTION` đang active, dùng access_token đã lưu (được refresh nền sẵn) để gọi API ngân hàng đối tác và tổng hợp số dư — mobile app vẫn chỉ thấy session_token nội bộ, không bao giờ chạm vào token của ngân hàng đối tác.

```mermaid
sequenceDiagram
    actor User
    participant MobileApp
    participant BFF as BFF Server
    participant DB as Database
    participant PartnerBank as Partner Bank API (Open Banking)

    User->>MobileApp: Mở màn hình số dư
    MobileApp->>BFF: GET /balance (session_token nội bộ)
    BFF->>DB: Lấy BANK_ACCOUNT chính và danh sách LINKED_BANK_CONNECTION active
    DB-->>BFF: tài khoản chính và các liên kết
    loop với mỗi ngân hàng đã liên kết
        BFF->>DB: Lấy access_token đã lưu (đã được refresh nền từ trước)
        BFF->>PartnerBank: GET /accounts/balance (access_token giữ ở BFF)
        PartnerBank-->>BFF: số dư của ngân hàng đó
    end
    BFF-->>MobileApp: Tổng hợp số dư nhiều ngân hàng
    MobileApp-->>User: Hiển thị số dư tổng hợp
    Note over MobileApp,BFF: Mobile app chỉ có session_token nội bộ, không bao giờ thấy access/refresh token của ngân hàng đối tác
```
