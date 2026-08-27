# Backend-for-Frontend (BFF) cho mobile app cần bảo mật cao

**Hệ thống:** App ngân hàng số (mobile-first) cần cho user liên kết tài khoản ngân hàng khác qua chuẩn Open Banking (dựa trên OAuth) để hiển thị tổng hợp số dư nhiều ngân hàng.

**Vai trò của flow:** Mobile app không tự giữ client_secret; phải qua một BFF server để thực hiện authorization code flow với PKCE, đảm bảo secret và token không bao giờ chạm tới thiết bị di động.

**Yêu cầu cụ thể:**
- Toàn bộ authorization code exchange (đổi code lấy token) phải diễn ra ở BFF server, mobile app chỉ nhận về session token nội bộ của chính hệ thống — không bao giờ thấy access/refresh token của ngân hàng đối tác.
- PKCE code_verifier phải được sinh và giữ ở mobile app, code_challenge gửi lên BFF, đúng chuẩn để chống bị đánh cắp authorization code qua custom URL scheme.
- Redirect sau khi user xác thực ở app ngân hàng đối tác phải quay lại đúng app mobile qua deep link/universal link, xử lý được trường hợp OS mở nhầm sang trình duyệt thường.
- Refresh token của từng ngân hàng liên kết phải được BFF lưu và tự refresh nền, mobile app không cần biết thời điểm hết hạn.
- Phải log được đầy đủ audit trail (ai liên kết ngân hàng nào, khi nào, từ thiết bị nào) để đáp ứng yêu cầu compliance của Open Banking.
