# Sequence — Login with Google

**Lớp: black-box** — **Nhóm: Thay đổi**. Diagram này thể hiện action "Login with Google" đã liệt kê ở Use case thay đổi — full authorization code flow, gồm state CSRF check, redirect_uri whitelist, domain restriction, và account linking đúng như yêu cầu đề bài.

```mermaid
sequenceDiagram
    actor U as Employee
    participant FE as Dashboard Frontend
    participant AUTH as Auth Service
    participant G as Google (OAuth Provider)
    participant DB as User DB

    U->>FE: Bấm "Sign in with Google"
    FE->>AUTH: GET /auth/google/start
    AUTH->>AUTH: Sinh state ngẫu nhiên, lưu vào session tạm
    AUTH-->>FE: Redirect tới Google authorize URL<br/>(client_id, redirect_uri, state, scope)
    FE->>G: Redirect trình duyệt tới Google
    G-->>U: Hiện màn hình consent
    U->>G: Bấm "Allow"
    G-->>FE: Redirect về redirect_uri?code=...&state=...
    FE->>AUTH: GET /auth/google/callback?code&state

    AUTH->>AUTH: Validate state khớp giá trị đã lưu (chống CSRF)
    AUTH->>AUTH: Validate redirect_uri khớp whitelist

    alt state không khớp hoặc redirect_uri không hợp lệ
        AUTH-->>FE: 400 Bad Request
        FE-->>U: Hiện lỗi, huỷ đăng nhập
    else hợp lệ
        AUTH->>G: POST /token {code, client_secret}
        G-->>AUTH: access_token (dùng 1 lần, không lưu lại)
        AUTH->>G: GET /userinfo (Authorization: access_token)
        G-->>AUTH: profile {email, domain, sub}

        alt email không thuộc domain công ty
            AUTH-->>FE: 403 Forbidden "domain không được phép"
            FE-->>U: Hiện lỗi, không tạo session
        else domain hợp lệ
            AUTH->>DB: SELECT user WHERE email = profile.email
            alt user đã tồn tại (email trùng account cũ)
                DB-->>AUTH: user row
                AUTH->>DB: INSERT oauth_identity {user_id, provider='google', sub}
            else user chưa tồn tại
                AUTH->>DB: INSERT user {email, password_hash=NULL}
                AUTH->>DB: INSERT oauth_identity {user_id, provider='google', sub}
            end
            AUTH->>AUTH: issue session token (JWT/cookie) — bỏ access_token của Google
            AUTH-->>FE: 200 OK {session token}
            FE-->>U: Redirect vào dashboard
        end
    end
```
