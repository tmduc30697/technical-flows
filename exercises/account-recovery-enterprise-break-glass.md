# Hệ thống nội bộ doanh nghiệp với "break-glass" recovery do helpdesk xử lý

**Hệ thống:** Một hệ thống quản lý tài liệu nội bộ nhạy cảm (hợp đồng, hồ sơ pháp lý) của doanh nghiệp, chính sách an ninh không cho phép self-service reset mật khẩu.

**Vai trò của flow:** Toàn bộ khôi phục tài khoản phải qua helpdesk/IT admin xử lý thủ công theo quy trình xác minh danh tính nội bộ, không có luồng tự động "quên mật khẩu" công khai.

**Yêu cầu cụ thể:**
- Không cung cấp bất kỳ endpoint tự phục vụ nào để reset mật khẩu — mọi yêu cầu phải đi qua ticket helpdesk gắn với xác minh danh tính nhân viên (ví dụ qua quản lý trực tiếp xác nhận).
- Khi helpdesk thực hiện reset cho một nhân viên, hệ thống phải buộc nhân viên đó đổi mật khẩu tạm thời ngay lần đăng nhập kế tiếp, không cho dùng mật khẩu tạm lâu dài.
- Mọi thao tác reset của helpdesk phải yêu cầu chính helpdesk agent đó xác thực MFA trước khi thực hiện, và ghi log agent nào đã reset cho tài khoản nào.
- Tài khoản có quyền cao (ví dụ admin hệ thống tài liệu) phải yêu cầu hai người xác nhận (four-eyes principle) trước khi được reset, không cho một agent helpdesk đơn lẻ tự thực hiện.
- Có cơ chế phát hiện bất thường nếu một agent helpdesk thực hiện reset cho số lượng tài khoản lớn bất thường trong thời gian ngắn (dấu hiệu agent bị mạo danh hoặc lạm quyền).
