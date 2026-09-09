# Enhance sequence — Detect suspicious enrollment (nhiều lần enroll liên tiếp, cảnh báo + khóa tạm thời)

Đây là **enhance**, flow mới phát sinh từ đề bài — hệ thống theo dõi tần suất enroll security key mới, nếu phát hiện nhiều lần enroll liên tiếp trong thời gian ngắn cho cùng 1 tài khoản (dấu hiệu account bị chiếm quyền), tự động cảnh báo và khóa tạm thời. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (đã chiếm được session)
    participant Server
    participant DB as Database
    actor SecurityTeam as Đội bảo mật

    Attacker->>Server: Enroll security key mới lần 1 (thời điểm t)
    Server->>DB: INSERT SECURITY_KEY, log enrolled_at=t
    DB-->>Server: OK

    Attacker->>Server: Enroll security key mới lần 2 (thời điểm t+30s)
    Server->>DB: INSERT SECURITY_KEY, log enrolled_at=t+30s
    DB-->>Server: OK

    Attacker->>Server: Enroll security key mới lần 3 (thời điểm t+50s)
    Server->>DB: SELECT COUNT(*) SECURITY_KEY WHERE user=... AND enrolled_at trong 5 phút gần nhất
    DB-->>Server: count=3 lần trong dưới 1 phút

    Note over Server: Vượt ngưỡng bất thường (vd quá 2 lần enroll trong 5 phút)
    Server->>DB: INSERT SECURITY_ALERT (type=rapid_enrollment, triggered_at=now(), account_locked_until=now()+30 phút)
    Server->>DB: UPDATE USER SET account_locked_until=now()+30 phút
    DB-->>Server: OK

    Server-->>Attacker: Từ chối enroll thêm, tài khoản tạm khóa 30 phút
    Server->>SecurityTeam: Gửi cảnh báo real-time, "Tài khoản SRE có dấu hiệu bị chiếm quyền, 3 lần enroll security key trong 1 phút"

    SecurityTeam->>Server: Xác minh và revoke toàn bộ security key mới enroll trong đợt này nếu xác nhận là tấn công
```
