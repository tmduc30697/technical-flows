# Enhance sequence — Post-recovery session reset & cooling-off

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 4 của đề bài: sau khi `RECOVERY_CASE` được duyệt (nối tiếp flow "Lost-channels recovery"), toàn bộ session/thiết bị cũ phải bị đăng xuất, và có cooling-off period trước khi cho rút tiền lớn lần đầu.

```mermaid
sequenceDiagram
    participant DB as RECOVERY_CASE store
    participant SessionStore as SESSION store
    participant Restriction as ACCOUNT_RESTRICTION store
    participant Audit as AUDIT_LOG
    actor User

    DB->>SessionStore: RECOVERY_CASE approved → tìm toàn bộ SESSION đang active của user
    SessionStore->>SessionStore: Đặt revoked_at cho mọi session cũ (mọi thiết bị)
    SessionStore->>Audit: Ghi log revoke toàn bộ session (immutable)
    DB->>Restriction: Gỡ ACCOUNT_RESTRICTION (withdrawal_frozen + view_only)
    DB->>Restriction: Tạo ACCOUNT_RESTRICTION mới (type=cooling_off_withdrawal_limit, active_from=now, active_until=now+24-48h)
    Restriction->>Audit: Ghi log áp dụng cooling-off (immutable)
    DB-->>User: Yêu cầu đăng nhập lại trên thiết bị mới (kênh liên hệ đã được cập nhật trong lúc xác minh)
    User->>User: Trong 24-48h: xem được tài khoản, giao dịch nhỏ bình thường, nhưng rút tiền lớn bị chặn
    Note over Restriction: Sau khi cooling_off_withdrawal_limit hết hạn, tài khoản trở lại hoạt động bình thường
```
