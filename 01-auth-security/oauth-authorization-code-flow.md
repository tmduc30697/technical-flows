# OAuth Authorization Code Flow — Đề bài thực hành

Các đề bài dưới đây đi từ vai trò **OAuth Client** (tiêu thụ đăng nhập/API bên thứ ba) đến vai trò **Authorization Server** (tự cấp quyền cho app thứ ba), nhằm luyện đủ các góc của authorization code flow trong bối cảnh web app.

---

## "Đăng nhập bằng Google" cho một SaaS dashboard nội bộ

**Repository:** `oauth-google-login-internal-dashboard`

**Hệ thống:** Một dashboard quản lý task nội bộ (kiểu Trello mini) cho công ty, hiện chỉ có đăng nhập bằng email/password.

**Vai trò của flow:** Thêm nút "Sign in with Google" — app đóng vai trò OAuth Client, dùng authorization code flow để lấy identity token và định danh user, thay thế/song song với login truyền thống.

**Yêu cầu cụ thể:**
- Chỉ cho phép đăng nhập bằng các Google account thuộc domain công ty (chặn domain khác).
- Lần đầu login qua Google phải tự tạo user record và map với account cũ nếu email đã tồn tại (account linking).
- Phải dùng `state` param để chống CSRF, và validate `redirect_uri` khớp whitelist.
- Access token của Google chỉ dùng một lần để lấy profile — không lưu lại; hệ thống tự phát session token riêng (JWT/session cookie) sau khi xác thực xong.
- Xử lý được case user bấm "Deny" ở màn hình consent của Google.

---

## Tự xây dựng OAuth Provider cho hệ sinh thái "app thứ ba"

**Repository:** `oauth-provider-third-party-ecosystem`

**Hệ thống:** Một platform e-commerce (giống Shopify) muốn cho phép các developer bên ngoài xây dựng app cắm vào (app store), cần cấp quyền truy cập API dữ liệu shop cho các app đó.

**Vai trò của flow:** Lần này hệ thống đóng vai trò **Authorization Server**, không phải Client — phải tự implement authorization endpoint, consent screen, token endpoint, và quản lý client_id/client_secret cho từng app đăng ký.

**Yêu cầu cụ thể:**
- Có cổng "Developer portal" để đăng ký OAuth app, nhận về `client_id`/`client_secret`, và khai báo `redirect_uri` whitelist.
- Authorization endpoint phải hiển thị consent screen liệt kê rõ scope app xin (ví dụ: đọc order, đọc customer, ghi inventory) và cho user chọn approve từng phần hoặc toàn bộ.
- Authorization code phát ra chỉ dùng được một lần, hết hạn trong vòng 60 giây, và phải validate PKCE (code_verifier/code_challenge) cho các app không giữ được client_secret an toàn (SPA/mobile).
- Access token phát ra phải mang theo scope đã được approve, và API resource server phải chặn request nếu access token không có đủ scope cần thiết cho endpoint đó.
- Có cơ chế cho user vào "App đã kết nối" để revoke quyền của một app bất kỳ, và revoke phải làm mọi token/refresh token liên quan hết hiệu lực ngay lập tức (không chờ token tự hết hạn).

---

## Backend-for-Frontend (BFF) cho mobile app cần bảo mật cao

**Repository:** `oauth-bff-mobile-secure`

**Hệ thống:** App ngân hàng số (mobile-first) cần cho user liên kết tài khoản ngân hàng khác qua chuẩn Open Banking (dựa trên OAuth) để hiển thị tổng hợp số dư nhiều ngân hàng.

**Vai trò của flow:** Mobile app không tự giữ client_secret; phải qua một BFF server để thực hiện authorization code flow với PKCE, đảm bảo secret và token không bao giờ chạm tới thiết bị di động.

**Yêu cầu cụ thể:**
- Toàn bộ authorization code exchange (đổi code lấy token) phải diễn ra ở BFF server, mobile app chỉ nhận về session token nội bộ của chính hệ thống — không bao giờ thấy access/refresh token của ngân hàng đối tác.
- PKCE code_verifier phải được sinh và giữ ở mobile app, code_challenge gửi lên BFF, đúng chuẩn để chống bị đánh cắp authorization code qua custom URL scheme.
- Redirect sau khi user xác thực ở app ngân hàng đối tác phải quay lại đúng app mobile qua deep link/universal link, xử lý được trường hợp OS mở nhầm sang trình duyệt thường.
- Refresh token của từng ngân hàng liên kết phải được BFF lưu và tự refresh nền, mobile app không cần biết thời điểm hết hạn.
- Phải log được đầy đủ audit trail (ai liên kết ngân hàng nào, khi nào, từ thiết bị nào) để đáp ứng yêu cầu compliance của Open Banking.
