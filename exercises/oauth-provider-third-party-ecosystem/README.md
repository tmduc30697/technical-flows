# Tự xây dựng OAuth Provider cho hệ sinh thái "app thứ ba"

**Hệ thống:** Một platform e-commerce (giống Shopify) muốn cho phép các developer bên ngoài xây dựng app cắm vào (app store), cần cấp quyền truy cập API dữ liệu shop cho các app đó.

**Vai trò của flow:** Lần này hệ thống đóng vai trò **Authorization Server**, không phải Client — phải tự implement authorization endpoint, consent screen, token endpoint, và quản lý client_id/client_secret cho từng app đăng ký.

**Yêu cầu cụ thể:**
- Có cổng "Developer portal" để đăng ký OAuth app, nhận về `client_id`/`client_secret`, và khai báo `redirect_uri` whitelist.
- Authorization endpoint phải hiển thị consent screen liệt kê rõ scope app xin (ví dụ: đọc order, đọc customer, ghi inventory) và cho user chọn approve từng phần hoặc toàn bộ.
- Authorization code phát ra chỉ dùng được một lần, hết hạn trong vòng 60 giây, và phải validate PKCE (code_verifier/code_challenge) cho các app không giữ được client_secret an toàn (SPA/mobile).
- Access token phát ra phải mang theo scope đã được approve, và API resource server phải chặn request nếu access token không có đủ scope cần thiết cho endpoint đó.
- Có cơ chế cho user vào "App đã kết nối" để revoke quyền của một app bất kỳ, và revoke phải làm mọi token/refresh token liên quan hết hiệu lực ngay lập tức (không chờ token tự hết hạn).
