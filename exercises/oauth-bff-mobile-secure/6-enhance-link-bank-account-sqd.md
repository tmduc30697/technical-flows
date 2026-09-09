# Enhance sequence — Link bank account

Đây là **enhance**, flow hoàn toàn mới so với base — user liên kết 1 ngân hàng đối tác qua Open Banking OAuth. Đây là flow trung tâm của enhance, thể hiện đầy đủ 4 yêu cầu: authorization code exchange chỉ diễn ra ở BFF, PKCE sinh ở mobile app, redirect quay lại đúng app qua deep link/universal link, và ghi audit trail đầy đủ.

```mermaid
sequenceDiagram
    actor User
    participant MobileApp
    participant BFF as BFF Server
    participant DB as Database
    participant PartnerBank as Partner Bank (Open Banking OAuth)

    User->>MobileApp: Chọn "Liên kết ngân hàng khác"
    MobileApp->>MobileApp: Sinh code_verifier, tính code_challenge theo chuẩn PKCE
    MobileApp->>BFF: POST /link/start { partner_bank_id, code_challenge, device_id }
    BFF->>DB: Lưu OAUTH_LINK_REQUEST (state, code_challenge, user_id, device_id)
    BFF-->>MobileApp: authorization_url kèm state
    MobileApp->>PartnerBank: Mở authorization_url
    PartnerBank-->>User: Yêu cầu đăng nhập và đồng ý chia sẻ dữ liệu
    User->>PartnerBank: Xác thực và đồng ý
    PartnerBank-->>MobileApp: Redirect kèm authorization code qua universal link/deep link
    Note over MobileApp,PartnerBank: Nếu OS mở nhầm sang trình duyệt thường, universal link vẫn điều hướng lại đúng app
    MobileApp->>BFF: POST /link/callback { code, state, code_verifier }
    BFF->>DB: Đối chiếu state với OAUTH_LINK_REQUEST đã lưu trước đó
    BFF->>PartnerBank: POST /token { code, code_verifier, client_secret }
    Note over BFF,PartnerBank: Toàn bộ exchange code lấy token diễn ra ở BFF, client_secret không rời BFF
    PartnerBank-->>BFF: access_token, refresh_token
    BFF->>DB: Lưu LINKED_BANK_CONNECTION (token đã mã hoá)
    BFF->>DB: Ghi AUDIT_LOG (user, ngân hàng, thời điểm, device_id)
    BFF-->>MobileApp: 200 liên kết thành công
    MobileApp-->>User: Hiển thị ngân hàng vừa liên kết
    Note over MobileApp,BFF: Mobile app chỉ nhận kết quả thành công, không bao giờ thấy access/refresh token của ngân hàng đối tác
```
