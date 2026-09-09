# Enhance sequence — Token exchange

Đây là **enhance**, flow hoàn toàn mới so với base — app đổi authorization code lấy access token/refresh token ở token endpoint. Flow này thể hiện rõ việc validate code chỉ dùng 1 lần, đã hết hạn thì từ chối, và validate PKCE `code_verifier` khớp với `code_challenge` đã lưu ở bước authorize.

```mermaid
sequenceDiagram
    participant App as Third-party App
    participant AuthServer
    participant DB as Database

    App->>AuthServer: POST /oauth/token { code, code_verifier, client_id, redirect_uri }
    AuthServer->>DB: Tìm AUTHORIZATION_CODE theo code
    alt code không tồn tại, đã dùng, hoặc quá 60 giây
        AuthServer-->>App: 400 invalid_grant
    else code còn hợp lệ
        AuthServer->>AuthServer: Kiểm tra SHA256(code_verifier) khớp code_challenge đã lưu
        alt PKCE không khớp
            AuthServer-->>App: 400 invalid_grant, PKCE verification failed
        else PKCE khớp
            AuthServer->>DB: Đánh dấu AUTHORIZATION_CODE used = true
            AuthServer->>DB: Tạo ACCESS_TOKEN (scopes từ APP_GRANT, expires_at ngắn hạn)
            AuthServer->>DB: Tạo REFRESH_TOKEN tương ứng
            DB-->>AuthServer: xác nhận đã lưu
            AuthServer-->>App: 200 { access_token, refresh_token, scope, expires_in }
        end
    end
    Note over AuthServer,DB: Authorization code dùng lần 2 sẽ luôn bị từ chối dù chưa hết hạn
```
