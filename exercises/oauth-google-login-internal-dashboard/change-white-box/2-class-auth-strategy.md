# Class — Auth Strategy (Email/Password vs Google OAuth)

**Lớp: white-box** — **Nhóm: Thay đổi**. Diagram này tự thiết kế breakdown OOP cho `AuthService`: khi có 2 cách đăng nhập cùng tồn tại song song (đề bài: "thay thế/song song với login truyền thống"), mỗi cách có logic xác thực khác hẳn nhau (so password hash vs. cả chuỗi OAuth code-exchange + domain check + account-linking) — Strategy pattern làm rõ ranh giới này và cho phép thêm provider OAuth khác sau này mà không sửa `AuthService`.

```mermaid
classDiagram
    class AuthStrategy {
        <<interface>>
        +authenticate(request) AuthResult
    }

    class EmailPasswordStrategy {
        +authenticate(request) AuthResult
        -verifyPassword(user, password) bool
    }

    class GoogleOAuthStrategy {
        +authenticate(request) AuthResult
        -validateState(state) bool
        -validateRedirectUri(uri) bool
        -exchangeCodeForToken(code) GoogleToken
        -fetchProfile(token) GoogleProfile
        -isCompanyDomain(email) bool
        -linkOrCreateUser(profile) User
    }

    class AuthService {
        -strategies: Map~string, AuthStrategy~
        +login(method, request) AuthResult
    }

    class SessionIssuer {
        +issue(user) SessionToken
    }

    class User {
        +id
        +email
        +password_hash
    }

    class OAuthIdentity {
        +id
        +user_id
        +provider
        +provider_user_id
    }

    AuthStrategy <|.. EmailPasswordStrategy
    AuthStrategy <|.. GoogleOAuthStrategy
    AuthService --> AuthStrategy : delegates theo method
    AuthService --> SessionIssuer : phát session sau khi authenticate thành công
    EmailPasswordStrategy --> User : verify password_hash
    GoogleOAuthStrategy --> User : link hoặc tạo mới
    GoogleOAuthStrategy --> OAuthIdentity : tạo bản ghi liên kết
```

**Vì sao Strategy chứ không if/else:** `GoogleOAuthStrategy` có state riêng (CSRF `state`, redirect_uri whitelist) mà `EmailPasswordStrategy` không có — nhét chung vào 1 class sẽ lẫn lộn field/method không dùng chung; tách class cũng khiến `AuthService.login()` không phình to khi có thêm provider OAuth thứ 3 sau này.
