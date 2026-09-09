# Enhance sequence — Revoke security key (thu hồi 1 key bị mất, không ảnh hưởng key còn lại)

Đây là **enhance**, flow mới phát sinh từ đề bài — khi 1 security key bị mất/đánh cắp, kỹ sư hoặc admin cần revoke ngay riêng key đó, các key khác của cùng tài khoản vẫn hoạt động bình thường. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor SRE as Kỹ sư SRE (vừa làm mất key chính)
    participant Server
    participant DB as Database

    SRE->>Server: Báo mất security key (credential_id_1, label=chính)
    Server->>DB: SELECT SECURITY_KEY WHERE user=SRE
    DB-->>Server: 2 key, credential_id_1 (chính, còn hiệu lực), credential_id_2 (dự phòng, còn hiệu lực)

    Server->>DB: UPDATE SECURITY_KEY SET revoked_at=now() WHERE credential_id=credential_id_1
    DB-->>Server: OK

    Server->>DB: SELECT COUNT(*) SECURITY_KEY WHERE user=SRE AND revoked_at IS NULL
    DB-->>Server: count=1 (chỉ còn credential_id_2)
    Server-->>SRE: Đã revoke key chính, còn 1 key dự phòng hoạt động, khuyến nghị đăng ký thêm key mới để đủ lại 2 key

    Note over Server,DB: Revoke chỉ set revoked_at cho đúng credential_id_1, không đụng tới credential_id_2, mọi LOGIN_SESSION trước đó dùng credential_id_1 không bị hủy hồi tố

    SRE->>Server: Đăng nhập lại bằng key chính đã mất (nếu kẻ gian có key vật lý)
    Server->>DB: SELECT SECURITY_KEY WHERE credential_id=credential_id_1
    DB-->>Server: revoked_at đã set
    Server-->>SRE: Từ chối, key này đã bị thu hồi, không còn hiệu lực
```
