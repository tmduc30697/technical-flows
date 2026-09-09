# Enhance sequence — Authorize consent

Đây là **enhance**, flow hoàn toàn mới so với base — app thứ ba redirect user tới authorization endpoint, hệ thống hiển thị consent screen liệt kê rõ scope app xin, cho user chọn approve từng phần hoặc toàn bộ, rồi phát authorization code chỉ dùng 1 lần trong 60 giây, kèm `code_challenge` theo PKCE cho app không giữ được client_secret an toàn.

```mermaid
sequenceDiagram
    actor Owner as Shop Owner
    participant App as Third-party App
    participant AuthServer
    participant DB as Database

    App->>App: Sinh code_verifier, tính code_challenge (PKCE, cho SPA/mobile)
    App->>Owner: Redirect tới authorization endpoint kèm client_id, scope, redirect_uri, code_challenge
    Owner->>AuthServer: GET /oauth/authorize (chưa login thì bắt login trước)
    AuthServer->>DB: Kiểm tra client_id hợp lệ, redirect_uri khớp whitelist đã đăng ký
    AuthServer-->>Owner: Hiển thị consent screen, liệt kê từng scope (đọc order, đọc customer, ghi inventory)
    Owner->>AuthServer: Chọn approve từng scope hoặc toàn bộ, bấm Allow
    AuthServer->>DB: Tạo APP_GRANT (user, shop, scopes đã approve)
    AuthServer->>DB: Tạo AUTHORIZATION_CODE (code_challenge, scopes, expires_at = +60s, used=false)
    DB-->>AuthServer: xác nhận đã lưu
    AuthServer-->>Owner: Redirect về redirect_uri kèm authorization code
    Owner-->>App: App nhận code qua redirect_uri
    Note over AuthServer,DB: Nếu user chỉ approve 1 phần scope, scope không được chọn sẽ không có trong code lẫn token sau này
```
