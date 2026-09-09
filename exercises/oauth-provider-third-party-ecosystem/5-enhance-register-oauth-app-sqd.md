# Enhance sequence — Register OAuth app

Đây là **enhance**, flow hoàn toàn mới so với base — developer bên ngoài đăng ký app qua Developer Portal, nhận về `client_id`/`client_secret` và khai báo `redirect_uri` whitelist. Đây là bước tiên quyết để bất kỳ app nào có thể tham gia flow authorization ở các bước sau.

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant Portal as Developer Portal
    participant AuthServer
    participant DB as Database

    Dev->>Portal: Đăng nhập Developer Portal
    Dev->>Portal: Tạo app mới, khai báo tên app và redirect_uri whitelist
    Portal->>AuthServer: POST /developer/apps { name, redirect_uris }
    AuthServer->>AuthServer: Sinh client_id, sinh client_secret
    AuthServer->>DB: Lưu OAUTH_APP (client_id, client_secret_hash, redirect_uris)
    DB-->>AuthServer: xác nhận đã lưu
    AuthServer-->>Portal: client_id, client_secret (chỉ hiển thị 1 lần)
    Portal-->>Dev: Hiển thị client_id/client_secret để lưu lại
    Note over Portal,Dev: client_secret không được hiển thị lại sau lần đầu tiên này
```
