# Sequence — Deny Google Consent

**Lớp: black-box** — **Nhóm: Thay đổi**. Diagram này thể hiện action "Deny Google consent" đã liệt kê ở Use case thay đổi (`<<extend>>` của "Login with Google") — đúng yêu cầu đề bài "xử lý được case user bấm Deny ở màn hình consent của Google". Luồng callback ở đây khác hẳn happy path: không có `code`, có `error=access_denied`, nên không thể gộp chung vào sequence "Login with Google".

```mermaid
sequenceDiagram
    actor U as Employee
    participant FE as Dashboard Frontend
    participant AUTH as Auth Service
    participant G as Google (OAuth Provider)

    U->>FE: Bấm "Sign in with Google"
    FE->>AUTH: GET /auth/google/start
    AUTH-->>FE: Redirect tới Google authorize URL (state, redirect_uri)
    FE->>G: Redirect trình duyệt tới Google
    G-->>U: Hiện màn hình consent
    U->>G: Bấm "Deny"
    G-->>FE: Redirect về redirect_uri?error=access_denied&state=...
    FE->>AUTH: GET /auth/google/callback?error=access_denied&state
    AUTH->>AUTH: Validate state khớp giá trị đã lưu
    AUTH->>AUTH: Phát hiện param "error" — không có "code" để đổi token
    AUTH-->>FE: 200 OK {status: "cancelled"} (không tạo session, không gọi Google token endpoint)
    FE-->>U: Hiện "Đăng nhập bị huỷ", cho phép quay lại chọn cách đăng nhập khác
```
