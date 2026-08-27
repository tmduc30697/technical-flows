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

---

## SaaS B2B cho phép admin công ty tự cấu hình chính sách MFA bắt buộc theo tổ chức

**Repository:** `mfa-b2b-saas-org-policy`

**Hệ thống:** Một SaaS quản lý dự án/CRM bán cho nhiều công ty khách hàng (multi-tenant), mỗi công ty có admin riêng quản lý chính sách bảo mật cho nhân viên của họ.

**Vai trò của flow:** Cho phép org admin tự cấu hình chính sách MFA bắt buộc (bật/tắt, phương thức được phép, áp dụng cho nhóm role nào) riêng cho tổ chức của họ, độc lập với các tenant khác trên cùng nền tảng.

**Yêu cầu cụ thể:**
- Khi admin bật chính sách bắt buộc MFA cho toàn tổ chức, những user đang có session hoạt động nhưng chưa enroll MFA không thể bị ép logout tức thì vì sẽ gây gián đoạn hàng loạt, nhưng cũng không thể được dùng vô thời hạn mà không enroll — cần một grace period có giới hạn rõ ràng kèm nhắc nhở tăng dần.
- Chính sách MFA có thể cấu hình khác nhau theo role trong cùng tổ chức (ví dụ chỉ bắt buộc cho admin/owner, không bắt buộc cho viewer) — khi một user được đổi role giữa chừng (từ viewer lên admin), chính sách MFA mới phải áp dụng ngay, không chờ tới lần đăng nhập kế tiếp mới phát hiện ra là chưa đủ điều kiện.
- Vì một tài khoản email có thể tham gia nhiều tổ chức trên cùng nền tảng với chính sách MFA khác nhau, cần định nghĩa rõ MFA áp dụng theo ngữ cảnh org nào trong phiên làm việc, tránh tình huống user né được yêu cầu MFA của tổ chức chặt hơn bằng cách chuyển sang truy cập qua context của tổ chức khác lỏng hơn.
- Khi admin hạ cấp hoặc tắt chính sách MFA bắt buộc (ví dụ do đổi nhà cung cấp SSO), đây là hành động làm giảm bảo mật toàn tổ chức nên cần yêu cầu chính admin đó xác thực MFA để thực hiện thay đổi, đồng thời ghi log và thông báo cho các admin khác trong tổ chức biết.
- Khi tổ chức bắt buộc một phương thức MFA cụ thể chặt hơn (ví dụ chỉ cho WebAuthn, cấm SMS OTP), những user đã enroll SMS theo chính sách cũ phải được yêu cầu enroll lại phương thức mới trong một mốc thời gian rõ ràng, không bị mất quyền truy cập đột ngột ngay khi chính sách vừa thay đổi.

---

## Mạng xã hội bảo vệ tài khoản creator/influencer lớn khỏi chiếm quyền và tấn công MFA fatigue

**Repository:** `mfa-social-creator-account-takeover-fatigue`

**Hệ thống:** Một nền tảng mạng xã hội cho phép đăng nội dung công khai, tài khoản có lượng follower lớn (creator/influencer) là mục tiêu thường xuyên bị nhắm tới để chiếm quyền, thường nhằm lừa đảo follower hoặc đòi tiền chuộc.

**Vai trò của flow:** Tự động áp dụng chính sách MFA mạnh hơn cho tài khoản có ảnh hưởng lớn, đồng thời phát hiện và ngăn chặn tấn công "MFA fatigue" — kẻ tấn công spam liên tục yêu cầu push notification approve nhằm khiến chủ tài khoản bấm nhầm chấp nhận.

**Yêu cầu cụ thể:**
- Hệ thống phải tự động nâng cấp yêu cầu MFA (ví dụ buộc chuyển từ SMS hoặc không có MFA sang WebAuthn/TOTP) khi tài khoản vượt một ngưỡng follower hoặc mức độ ảnh hưởng nhất định, kèm luồng thông báo và hỗ trợ enroll để không đột ngột khóa quyền truy cập của creator đang hoạt động bình thường.
- Phải giới hạn số lượng push notification MFA gửi liên tiếp trong một khoảng thời gian ngắn, tự động khóa các yêu cầu mới sau một số lần nhất định, để chặn kịch bản kẻ tấn công spam hàng chục request đăng nhập liên tục nhằm khiến chủ tài khoản mệt mỏi và bấm approve nhầm.
- Giao diện push notification approve phải hiển thị đủ ngữ cảnh (vị trí, thiết bị, thời gian request) và yêu cầu một hành động xác nhận có chủ đích, ví dụ nhập số hiển thị trên màn hình đăng nhập vào app push (number matching), thay vì chỉ một nút "Approve" đơn giản rất dễ bấm nhầm khi đang bị dồn dập.
- Khi phát hiện một chuỗi request MFA bị từ chối liên tiếp trong thời gian ngắn — dấu hiệu rõ ràng của tấn công thay vì nhầm lẫn ngẫu nhiên — hệ thống phải tự động tạm khóa đăng nhập từ nguồn phát sinh request và cảnh báo chủ tài khoản qua một kênh độc lập, không chờ họ tự nhận ra đang bị tấn công.
- Với tài khoản creator lớn, cần một phương thức phục hồi khẩn cấp riêng (đường dây hỗ trợ ưu tiên hoặc xác minh danh tính tăng cường) khi họ thực sự mất yếu tố MFA, cân bằng giữa tốc độ khôi phục — vì mất quyền truy cập lâu gây thiệt hại uy tín/doanh thu — và rủi ro kẻ tấn công lợi dụng chính kênh phục hồi ưu tiên này để chiếm tài khoản.
