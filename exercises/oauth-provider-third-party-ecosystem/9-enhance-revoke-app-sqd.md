# Enhance sequence — Revoke app

Đây là **enhance**, flow hoàn toàn mới so với base — user vào màn hình "App đã kết nối" để thu hồi quyền của 1 app bất kỳ. Revoke phải làm mọi access token/refresh token liên quan tới `APP_GRANT` đó hết hiệu lực ngay lập tức, không chờ tự hết hạn.

```mermaid
sequenceDiagram
    actor Owner as Shop Owner
    participant AdminDashboard
    participant AuthServer
    participant DB as Database

    Owner->>AdminDashboard: Mở màn hình "App đã kết nối"
    AdminDashboard->>AuthServer: GET /apps/connected (session_token)
    AuthServer->>DB: Lấy danh sách APP_GRANT active của user
    DB-->>AuthServer: danh sách app đã cấp quyền
    AuthServer-->>AdminDashboard: danh sách app kèm scope đã approve
    AdminDashboard-->>Owner: Hiển thị danh sách
    Owner->>AdminDashboard: Bấm "Revoke" ở 1 app cụ thể
    AdminDashboard->>AuthServer: POST /apps/connected/{app_grant_id}/revoke
    AuthServer->>DB: UPDATE app_grant SET status = revoked, revoked_at = now()
    AuthServer->>DB: UPDATE access_token SET revoked_at = now() WHERE app_grant_id = :id
    AuthServer->>DB: UPDATE refresh_token SET revoked_at = now() WHERE access_token_id IN (...)
    DB-->>AuthServer: xác nhận đã cập nhật toàn bộ
    AuthServer-->>AdminDashboard: 200 đã thu hồi quyền
    AdminDashboard-->>Owner: Hiển thị app đã bị revoke
    Note over AuthServer,DB: Access token cũ bị chặn ngay ở lần gọi API tiếp theo, không chờ expires_at tự nhiên
```
