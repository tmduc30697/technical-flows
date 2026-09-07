# Công cụ nội bộ DevOps bắt buộc hardware security key cho tài khoản đặc quyền

**Hệ thống:** Một bảng điều khiển nội bộ cho phép kỹ sư SRE/DevOps truy cập server production, secret, và pipeline deploy.

**Vai trò của flow:** MFA bằng WebAuthn/FIDO2 (hardware security key hoặc platform authenticator) là bắt buộc cho mọi tài khoản có quyền production, nhằm chống phishing thay vì chỉ dùng OTP có thể bị đánh cắp.

**Yêu cầu cụ thể:**
- Đăng nhập bằng phương thức MFA yếu hơn (SMS OTP) phải bị chặn hoàn toàn cho tài khoản có quyền production — không được coi là factor hợp lệ dù đã enroll trước đó.
- Quy trình enroll security key phải yêu cầu ít nhất 2 key được đăng ký (chính + dự phòng) trước khi tài khoản được cấp quyền production, đề phòng mất một key.
- Khi một security key bị mất/đánh cắp, phải có cách revoke ngay riêng key đó mà không ảnh hưởng tới các key khác đã đăng ký cho cùng tài khoản.
- Hệ thống phải kiểm tra origin/relying party ID đúng chuẩn WebAuthn để chống các cuộc tấn công giả mạo domain (phishing-resistant thực sự, không chỉ về tên gọi).
- Có cơ chế cảnh báo/khóa tạm thời khi phát hiện nhiều lần enroll security key mới liên tiếp trong thời gian ngắn cho cùng một tài khoản (dấu hiệu account bị chiếm quyền).
