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

## App tích hợp với cửa hàng Shopify của người dùng

**Repository:** `oauth-shopify-app-integration`

**Hệ thống:** Một SaaS phân tích doanh thu cho chủ shop online, cần đọc dữ liệu order/inventory từ shop Shopify của họ.

**Vai trò của flow:** App là OAuth Client xin quyền truy cập API Shopify theo scope hạn chế (chỉ đọc order, không được ghi), chạy song song cho nhiều tenant (mỗi user = một shop riêng).

**Yêu cầu cụ thể:**
- Mỗi user có thể kết nối/disconnect nhiều store Shopify khác nhau vào cùng một account.
- Access token + refresh token của mỗi store phải được mã hóa khi lưu DB, không log ra plaintext.
- Khi refresh token bị revoke từ phía Shopify (user rút quyền), hệ thống phải phát hiện ở lần gọi API kế tiếp và đánh dấu store là "cần kết nối lại" — không được crash job nền.
- Có trang "Connected accounts" cho user xem scope đã cấp và thời điểm cấp quyền.
- Test được case race condition: hai request đồng thời cùng dùng refresh token cũ để lấy access token mới (refresh token rotation).

---

## Nền tảng lịch hẹn đồng bộ hai chiều với Google Calendar

**Repository:** `oauth-google-calendar-two-way-sync`

**Hệ thống:** App đặt lịch hẹn (booking) cho freelancer, cần đồng bộ 2 chiều: đọc lịch bận của freelancer từ Google Calendar và ghi event mới khi có booking.

**Vai trò của flow:** OAuth Client xin `offline_access` để có refresh token dài hạn, phục vụ background job đồng bộ định kỳ mà không cần user online.

**Yêu cầu cụ thể:**
- Khi cấp quyền lần đầu, phải xin đúng scope `calendar.events` + `calendar.readonly`, hiển thị rõ cho user biết app sẽ đọc/ghi gì.
- Refresh token phải được dùng để tự động lấy access token mới cho job chạy mỗi 15 phút, không được yêu cầu user đăng nhập lại.
- Nếu access token hết hạn giữa lúc job đang chạy, job phải tự refresh và retry đúng 1 lần, không loop vô hạn.
- Có cơ chế phát hiện và tự tắt đồng bộ nếu refresh token bị Google reject liên tục (ví dụ user đổi password Google).
- Ghi log audit: mỗi lần app tạo/sửa event trên calendar của user phải lưu lại được ai/khi nào để phục vụ debug khi user thắc mắc.

---

## Trình quản lý mạng xã hội đa nền tảng (social scheduler)

**Repository:** `oauth-social-media-scheduler`

**Hệ thống:** App cho phép người dùng soạn 1 bài viết và đăng đồng thời lên nhiều nền tảng (Facebook Page, Twitter/X, LinkedIn).

**Vai trò của flow:** App phải chạy authorization code flow riêng biệt với 3 provider khác nhau, mỗi provider có cách cấp scope/token khác nhau, và quản lý được nhiều "connected identity" cho cùng một tính năng "post bài".

**Yêu cầu cụ thể:**
- Kiến trúc phải trừu tượng hóa được sự khác biệt giữa 3 OAuth provider (authorization endpoint, token endpoint, cách refresh token khác nhau) sau một interface chung.
- Cho phép một user kết nối nhiều Facebook Page cùng lúc (không chỉ 1 Page/user).
- Khi đăng bài thất bại vì token của 1 platform bị hết hạn/revoke, các platform khác vẫn phải đăng thành công — trả về báo cáo per-platform (partial success).
- Có UI hiển thị trạng thái kết nối (đang hoạt động/hết hạn/lỗi) theo từng platform, theo từng page/account.
- Xử lý callback URL bị nền tảng thứ 3 gọi lại không đúng thứ tự (user mở nhiều tab kết nối cùng lúc).

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
