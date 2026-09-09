# Enhance sequence — Background token refresh

Đây là **enhance**, flow hoàn toàn mới so với base — BFF tự refresh refresh_token của từng ngân hàng liên kết trước khi access_token hết hạn, chạy nền định kỳ. Đáp ứng yêu cầu "mobile app không cần biết thời điểm hết hạn" trong đề bài.

```mermaid
sequenceDiagram
    participant Scheduler as Background Scheduler
    participant BFF as BFF Server
    participant DB as Database
    participant PartnerBank as Partner Bank (Open Banking OAuth)

    Scheduler->>BFF: Kích hoạt job refresh token định kỳ
    BFF->>DB: Lấy các LINKED_BANK_CONNECTION sắp hết hạn access_token
    loop với mỗi connection sắp hết hạn
        BFF->>PartnerBank: POST /token { grant_type=refresh_token, refresh_token }
        PartnerBank-->>BFF: access_token mới, refresh_token mới nếu có rotate
        BFF->>DB: Cập nhật access_token, refresh_token, token_expires_at
        BFF->>DB: Ghi AUDIT_LOG event_type=token_refreshed
    end
    BFF-->>Scheduler: Job hoàn tất
    Note over BFF,DB: Mobile app không cần biết thời điểm hết hạn hay tự thực hiện refresh
```
