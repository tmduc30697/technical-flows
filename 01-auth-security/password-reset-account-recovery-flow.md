# Password reset & Account recovery flow — Đề bài thực hành

Các đề bài dưới đây đi từ luồng quên mật khẩu tiêu chuẩn qua email, đến các bối cảnh phức tạp hơn (xác thực bằng số điện thoại, tài khoản SSO-only, mất toàn bộ kênh liên hệ, yêu cầu compliance cao) trong marketplace, mobile app, B2B SaaS, fintech, doanh nghiệp nội bộ và y tế.

---

## Marketplace thương mại điện tử với quên mật khẩu qua email

**Repository:** `account-recovery-marketplace-email-reset`

**Hệ thống:** Một sàn thương mại điện tử tiêu dùng phổ thông, tài khoản đăng ký bằng email/password.

**Vai trò của flow:** Luồng quên mật khẩu tiêu chuẩn — user nhập email, nhận link reset, đặt mật khẩu mới — phải an toàn ở quy mô lớn và chống được các kiểu tấn công phổ biến.

**Yêu cầu cụ thể:**
- Không được để endpoint "quên mật khẩu" lộ thông tin email có tồn tại trong hệ thống hay không (chống account enumeration) — response phải giống nhau dù email tồn tại hay không.
- Token reset phải là random đủ mạnh, chỉ dùng được một lần, hết hạn trong khoảng thời gian ngắn (ví dụ 15-30 phút), và bị vô hiệu ngay khi user đặt mật khẩu mới thành công.
- Nếu user bấm nhiều lần vào nút "gửi lại link reset", chỉ token mới nhất còn hiệu lực — các token cũ phát trước đó phải bị vô hiệu ngay.
- Sau khi đổi mật khẩu thành công, phải tự động đăng xuất tất cả session đang hoạt động ở nơi khác (trừ thiết bị vừa thực hiện đổi), và gửi email thông báo "mật khẩu của bạn vừa được đổi" kèm cách báo cáo nếu không phải chính chủ.
- Rate-limit request reset password theo cả email và theo IP để chống spam gửi email hoặc brute-force dò token.

---

## Fintech: khôi phục tài khoản khi mất cả email và số điện thoại đăng ký

**Repository:** `account-recovery-fintech-lost-channels`

**Hệ thống:** Một app ví điện tử/đầu tư cá nhân, tài khoản có gắn với tiền thật, quy định pháp lý yêu cầu xác minh danh tính nghiêm ngặt.

**Vai trò của flow:** Xử lý case khó nhất — người dùng mất quyền truy cập cả email và số điện thoại đã đăng ký, cần một quy trình khôi phục có kiểm soát dựa trên xác minh danh tính, không thể tự động hoàn toàn.

**Yêu cầu cụ thể:**
- Khi không còn kênh liên hệ nào khớp với thông tin đã đăng ký, hệ thống phải chuyển sang luồng xác minh danh tính thủ công/bán tự động (ví dụ upload giấy tờ tùy thân, chụp ảnh khớp khuôn mặt) trước khi cấp lại quyền truy cập.
- Trong suốt thời gian chờ xác minh, tài khoản phải bị đóng băng các hành động rút tiền/chuyển tiền, nhưng vẫn có thể xem được thông tin (nếu cần) để tránh vừa mất quyền truy cập vừa mất khả năng theo dõi tài sản.
- Phải có audit trail đầy đủ và không thể chỉnh sửa cho toàn bộ quy trình xác minh khôi phục (ai duyệt, dựa trên bằng chứng gì, khi nào) để đáp ứng yêu cầu compliance tài chính khi bị thanh tra.
- Sau khi khôi phục thành công, mọi thiết bị/session cũ phải bị đăng xuất, và phải có một khoảng "cooling-off period" (ví dụ 24-48 giờ) trước khi cho phép rút tiền lớn lần đầu sau khôi phục, để giảm rủi ro nếu quy trình xác minh bị lợi dụng.
- Thiết kế được cơ chế chống social engineering nhằm vào bộ phận hỗ trợ khách hàng (kẻ tấn công giả làm chủ tài khoản để yêu cầu khôi phục) bằng việc yêu cầu nhiều bằng chứng độc lập, không dựa vào một câu hỏi bảo mật duy nhất.

---

## Hệ thống nội bộ doanh nghiệp với "break-glass" recovery do helpdesk xử lý

**Repository:** `account-recovery-enterprise-break-glass`

**Hệ thống:** Một hệ thống quản lý tài liệu nội bộ nhạy cảm (hợp đồng, hồ sơ pháp lý) của doanh nghiệp, chính sách an ninh không cho phép self-service reset mật khẩu.

**Vai trò của flow:** Toàn bộ khôi phục tài khoản phải qua helpdesk/IT admin xử lý thủ công theo quy trình xác minh danh tính nội bộ, không có luồng tự động "quên mật khẩu" công khai.

**Yêu cầu cụ thể:**
- Không cung cấp bất kỳ endpoint tự phục vụ nào để reset mật khẩu — mọi yêu cầu phải đi qua ticket helpdesk gắn với xác minh danh tính nhân viên (ví dụ qua quản lý trực tiếp xác nhận).
- Khi helpdesk thực hiện reset cho một nhân viên, hệ thống phải buộc nhân viên đó đổi mật khẩu tạm thời ngay lần đăng nhập kế tiếp, không cho dùng mật khẩu tạm lâu dài.
- Mọi thao tác reset của helpdesk phải yêu cầu chính helpdesk agent đó xác thực MFA trước khi thực hiện, và ghi log agent nào đã reset cho tài khoản nào.
- Tài khoản có quyền cao (ví dụ admin hệ thống tài liệu) phải yêu cầu hai người xác nhận (four-eyes principle) trước khi được reset, không cho một agent helpdesk đơn lẻ tự thực hiện.
- Có cơ chế phát hiện bất thường nếu một agent helpdesk thực hiện reset cho số lượng tài khoản lớn bất thường trong thời gian ngắn (dấu hiệu agent bị mạo danh hoặc lạm quyền).
