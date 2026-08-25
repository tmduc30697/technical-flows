# Multi-factor authentication (MFA) flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại yếu tố xác thực thứ hai (TOTP, SMS, push notification, biometric, hardware key/WebAuthn) trong các bối cảnh fintech, B2B SaaS, doanh nghiệp nội bộ, mạng xã hội và y tế, luyện cả enrollment, xác thực, step-up và account recovery khi mất yếu tố thứ hai.

---

## Ngân hàng số bắt buộc MFA cho đăng nhập và giao dịch nhạy cảm

**Repository:** `mfa-digital-bank-mandatory`

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền, thanh toán hóa đơn, xem sao kê.

**Vai trò của flow:** MFA bắt buộc ngay từ lần đăng nhập đầu tiên (TOTP app + SMS backup), và step-up MFA riêng cho các giao dịch có giá trị lớn hoặc thêm người thụ hưởng mới.

**Yêu cầu cụ thể:**
- Bắt buộc enroll ít nhất một phương thức MFA trước khi được sử dụng bất kỳ tính năng nào của app, không cho phép "bỏ qua để sau".
- Giao dịch vượt một ngưỡng số tiền hoặc thêm beneficiary mới phải yêu cầu xác thực MFA lại dù session đăng nhập còn hạn (step-up theo hành vi, không chỉ theo thời gian).
- Mã OTP (TOTP/SMS) chỉ dùng được một lần, hết hạn sau một khoảng thời gian ngắn, và phải giới hạn số lần thử sai trước khi tạm khóa xác thực.
- Nếu SMS gửi OTP thất bại hoặc chậm, phải có phương án fallback rõ ràng (ví dụ dùng TOTP app) mà không làm người dùng bị kẹt hoàn toàn ngoài tài khoản.
- Mọi thay đổi liên quan đến MFA (thêm/xóa thiết bị, đổi số điện thoại nhận OTP) phải tự nó yêu cầu xác thực MFA hiện tại, và gửi thông báo cảnh báo qua kênh khác (email) ngay khi thay đổi.

---

## Công cụ nội bộ DevOps bắt buộc hardware security key cho tài khoản đặc quyền

**Repository:** `mfa-devops-hardware-key-privileged`

**Hệ thống:** Một bảng điều khiển nội bộ cho phép kỹ sư SRE/DevOps truy cập server production, secret, và pipeline deploy.

**Vai trò của flow:** MFA bằng WebAuthn/FIDO2 (hardware security key hoặc platform authenticator) là bắt buộc cho mọi tài khoản có quyền production, nhằm chống phishing thay vì chỉ dùng OTP có thể bị đánh cắp.

**Yêu cầu cụ thể:**
- Đăng nhập bằng phương thức MFA yếu hơn (SMS OTP) phải bị chặn hoàn toàn cho tài khoản có quyền production — không được coi là factor hợp lệ dù đã enroll trước đó.
- Quy trình enroll security key phải yêu cầu ít nhất 2 key được đăng ký (chính + dự phòng) trước khi tài khoản được cấp quyền production, đề phòng mất một key.
- Khi một security key bị mất/đánh cắp, phải có cách revoke ngay riêng key đó mà không ảnh hưởng tới các key khác đã đăng ký cho cùng tài khoản.
- Hệ thống phải kiểm tra origin/relying party ID đúng chuẩn WebAuthn để chống các cuộc tấn công giả mạo domain (phishing-resistant thực sự, không chỉ về tên gọi).
- Có cơ chế cảnh báo/khóa tạm thời khi phát hiện nhiều lần enroll security key mới liên tiếp trong thời gian ngắn cho cùng một tài khoản (dấu hiệu account bị chiếm quyền).

---

## Nền tảng y tế từ xa với MFA thích ứng theo rủi ro (risk-based/adaptive)

**Repository:** `mfa-telehealth-risk-based`

**Hệ thống:** Một nền tảng khám bệnh từ xa (telemedicine) cho bác sĩ và bệnh nhân, phải tuân thủ quy định bảo mật dữ liệu y tế.

**Vai trò của flow:** MFA bắt buộc cho bác sĩ/nhân viên y tế, nhưng áp dụng adaptive MFA: chỉ yêu cầu xác thực thêm khi có tín hiệu rủi ro (thiết bị mới, vị trí bất thường, giờ truy cập lạ), giảm ma sát khi rủi ro thấp.

**Yêu cầu cụ thể:**
- Hệ thống phải tính điểm rủi ro dựa trên ít nhất: thiết bị đã biết hay chưa, vị trí địa lý (so với lịch sử đăng nhập), và thời điểm truy cập, để quyết định có yêu cầu MFA lần này hay không.
- Với bệnh nhân xem hồ sơ bệnh án cá nhân, MFA chỉ bắt buộc khi truy cập từ thiết bị/vị trí chưa từng ghi nhận; với bác sĩ truy cập hồ sơ nhiều bệnh nhân, MFA bắt buộc mỗi phiên làm việc mới không phân biệt rủi ro.
- Khi điểm rủi ro cao bất thường (ví dụ đăng nhập từ hai quốc gia cách nhau vài phút), phải chặn và yêu cầu xác minh bổ sung mạnh hơn (ví dụ liên hệ hỗ trợ) thay vì chỉ hỏi lại OTP thông thường.
- Toàn bộ quyết định yêu cầu/miễn MFA của mỗi lần đăng nhập phải được log lại kèm lý do (rủi ro thấp/cao, tín hiệu nào kích hoạt) để phục vụ audit compliance.
- Nếu dịch vụ tính điểm rủi ro gặp lỗi hoặc timeout, hệ thống phải fail-safe về yêu cầu MFA đầy đủ, không được mặc định bỏ qua MFA vì lỗi hạ tầng.
