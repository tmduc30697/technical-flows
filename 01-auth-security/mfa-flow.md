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

## B2B SaaS với MFA tùy chọn cho user thường, bắt buộc cho admin

**Repository:** `mfa-b2b-saas-admin-required`

**Hệ thống:** Một SaaS quản lý bán hàng (CRM) cho doanh nghiệp nhỏ, có nhiều role: sales rep, manager, account owner/admin.

**Vai trò của flow:** MFA tùy chọn cho user thường nhưng bắt buộc cho role admin/owner (những người có quyền quản lý billing, user, tích hợp API).

**Yêu cầu cụ thể:**
- Hệ thống phải tự phát hiện khi một user được nâng lên role admin và bắt họ enroll MFA trước khi được giữ nguyên quyền admin (không cho "admin không MFA" tồn tại).
- Cung cấp recovery code (một bộ mã dùng một lần) khi enroll MFA, cảnh báo rõ user phải lưu ở nơi an toàn, và cho phép dùng đúng một lần mỗi mã.
- Admin công ty có thể cấu hình policy bắt buộc MFA cho toàn bộ nhân viên trong tenant của họ, và hệ thống phải chặn đăng nhập của user chưa enroll khi policy này được bật.
- Khi user mất cả thiết bị TOTP và toàn bộ recovery code, phải có quy trình khôi phục có kiểm soát (ví dụ xác minh qua admin khác trong công ty) thay vì tự động disable MFA.
- Test race condition: hai request đồng thời cùng dùng một recovery code — chỉ một được chấp nhận, request sau phải bị từ chối dù trong cùng millisecond.

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

## Mạng xã hội triển khai MFA dạng push notification cho toàn bộ user

**Repository:** `mfa-social-push-notification`

**Hệ thống:** Một mạng xã hội với hàng triệu người dùng, muốn tăng bảo mật bằng MFA dạng push notification (giống Duo/Authy) qua app đồng hành trên điện thoại.

**Vai trò của flow:** Cho phép người dùng phổ thông (không rành kỹ thuật) enroll và dùng MFA một cách thuận tiện, đồng thời có phương án phục hồi khi mất điện thoại — trọng tâm là trải nghiệm ở quy mô lớn.

**Yêu cầu cụ thể:**
- Khi đăng nhập từ thiết bị mới, gửi push notification tới thiết bị đã đăng ký kèm thông tin ngữ cảnh (vị trí ước lượng, loại thiết bị đang đăng nhập) để user nhận biết đúng/sai request.
- Push request phải tự hết hạn sau một khoảng thời gian ngắn nếu user không phản hồi, tránh tồn đọng request cũ có thể bị approve nhầm sau này.
- Phải chống được "MFA fatigue attack" (kẻ tấn công gửi liên tục nhiều push request hy vọng user bấm chấp nhận nhầm) bằng rate limiting và yêu cầu nhập số khớp (number matching) thay vì chỉ bấm "Approve".
- Cho phép enroll nhiều thiết bị nhận push cùng lúc (ví dụ điện thoại chính + máy tính bảng), và revoke được từng thiết bị riêng lẻ.
- Quy trình phục hồi khi mất điện thoại phải xác minh danh tính qua ít nhất hai tín hiệu độc lập khác (ví dụ email đã xác minh lâu năm + trả lời thông tin tài khoản) trước khi cho enroll lại thiết bị MFA mới, để tránh social engineering.

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

---

## App ngân hàng di động dùng biometric làm yếu tố thứ hai

**Repository:** `mfa-mobile-banking-biometric`

**Hệ thống:** Một app ngân hàng thuần di động (mobile-only), người dùng đăng nhập bằng password rồi xác thực thêm bằng Face ID/fingerprint ngay trên thiết bị.

**Vai trò của flow:** Biometric đóng vai trò yếu tố thứ hai gắn với thiết bị cụ thể, cần fallback OTP khi biometric không khả dụng và xử lý khi người dùng đổi thiết bị.

**Yêu cầu cụ thể:**
- Dữ liệu biometric không được rời khỏi thiết bị (chỉ dùng secure enclave/keystore của hệ điều hành để ký một challenge từ server) — server không bao giờ lưu hoặc nhận dữ liệu sinh trắc học thô.
- Khi người dùng đổi điện thoại mới, phải yêu cầu xác thực lại bằng phương thức khác (OTP qua SMS/email đã xác minh trước) trước khi cho đăng ký biometric trên thiết bị mới, và tự động vô hiệu binding biometric cũ trên thiết bị cũ.
- Nếu biometric thất bại nhiều lần liên tiếp (do vân tay/khuôn mặt không khớp), phải fallback về nhập OTP mà không tự động khóa tài khoản chỉ vì lỗi phần cứng đọc biometric.
- Phát hiện và chặn trường hợp app chạy trên thiết bị đã root/jailbreak cố gắng giả lập kết quả xác thực biometric thành công.
- Ghi log mỗi lần binding/unbinding biometric với một thiết bị (thời gian, model thiết bị) để người dùng có thể tự xem lại trong phần "Thiết bị đã xác thực" của app.
