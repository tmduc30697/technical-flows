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

## App di động (đặt xe/giao hàng) với tài khoản định danh bằng số điện thoại

**Repository:** `account-recovery-mobile-phone-identity`

**Hệ thống:** Một app đặt xe/giao hàng mà tài khoản người dùng được định danh chính bằng số điện thoại, không bắt buộc email.

**Vai trò của flow:** Khôi phục truy cập tài khoản khi người dùng đổi điện thoại hoặc quên mật khẩu, dựa trên xác thực OTP qua SMS tới số điện thoại đã đăng ký.

**Yêu cầu cụ thể:**
- OTP gửi qua SMS phải giới hạn số lần gửi trong một khoảng thời gian (ví dụ tối đa 5 lần/giờ) và giới hạn số lần nhập sai trước khi tạm khóa yêu cầu OTP cho số đó.
- Phải xử lý được trường hợp số điện thoại đã đăng ký bị chuyển sang chủ sở hữu mới (SIM recycling) — không cho phép người dùng mới của số điện thoại đó tự động chiếm được tài khoản cũ chỉ bằng OTP.
- Nếu người dùng đổi thiết bị mới nhưng vẫn giữ số điện thoại, luồng khôi phục phải phân biệt được "đổi thiết bị hợp lệ" và "kẻ tấn công có SIM nhưng không phải chủ tài khoản" bằng ít nhất một tín hiệu xác minh bổ sung.
- Toàn bộ session cũ trên thiết bị trước phải bị vô hiệu khi có yêu cầu khôi phục thành công trên thiết bị mới, và gửi cảnh báo qua kênh khác (email phụ nếu có, hoặc thông báo push tới thiết bị cũ nếu vẫn còn mạng) trước khi hoàn tất chuyển giao.
- Có cơ chế phát hiện pattern bất thường (nhiều tài khoản khác nhau cùng yêu cầu OTP dồn dập tới các số điện thoại liên tiếp nhau) để chặn tấn công tự động hàng loạt.

---

## B2B SaaS với tài khoản hỗn hợp: một phần SSO, một phần password

**Repository:** `account-recovery-b2b-saas-hybrid-sso`

**Hệ thống:** Một SaaS quản lý nhân sự cho doanh nghiệp, một số công ty khách hàng bắt buộc nhân viên đăng nhập qua SSO của công ty, một số công ty khác vẫn dùng email/password truyền thống.

**Vai trò của flow:** Luồng quên mật khẩu/khôi phục phải phân biệt đúng loại tài khoản và điều hướng phù hợp, tránh gây nhầm lẫn hoặc tạo lỗ hổng bỏ qua SSO.

**Yêu cầu cụ thể:**
- Khi user thuộc công ty bắt buộc SSO nhập email vào form "quên mật khẩu", hệ thống phải từ chối tạo token reset và hướng dẫn họ liên hệ SSO/admin công ty — không được vô tình cho phép tạo mật khẩu cục bộ (điều này sẽ phá vỡ chính sách bắt buộc SSO).
- Với công ty dùng password truyền thống, luồng reset chuẩn (token một lần, hết hạn ngắn) áp dụng như bình thường.
- Admin công ty phải có quyền tự "force reset" mật khẩu của một nhân viên cụ thể trong tenant của họ (ví dụ khi nhân viên nghỉ việc gấp), tách biệt khỏi self-service reset của user.
- Ghi log riêng cho mọi hành động liên quan tới reset/force-reset mật khẩu (ai thực hiện, cho ai, khi nào) để phục vụ audit nội bộ của từng công ty khách hàng.
- Nếu một công ty chuyển đổi chính sách (từ password sang bắt buộc SSO), các token reset password đang tồn tại (nếu có) của nhân viên công ty đó phải bị vô hiệu ngay khi chính sách mới có hiệu lực.

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

---

## Cổng bệnh nhân y tế với xác minh danh tính nhiều yếu tố khi khôi phục

**Repository:** `account-recovery-patient-portal-identity-verification`

**Hệ thống:** Một cổng thông tin cho bệnh nhân xem kết quả xét nghiệm, lịch hẹn, và trao đổi với bác sĩ.

**Vai trò của flow:** Khôi phục tài khoản khi bệnh nhân quên mật khẩu phải xác minh danh tính chặt hơn thông thường do dữ liệu là thông tin y tế nhạy cảm, cân bằng giữa an toàn và việc bệnh nhân (thường không rành công nghệ) vẫn dùng được.

**Yêu cầu cụ thể:**
- Luồng khôi phục phải kết hợp ít nhất hai yếu tố xác minh độc lập trong số: email/SMS đã đăng ký, ngày sinh, và một thông tin chỉ hệ thống y tế biết (ví dụ mã số bệnh nhân) — không chỉ dựa vào một email link đơn thuần như ứng dụng thông thường.
- Nếu bệnh nhân nhập sai thông tin xác minh quá số lần cho phép, phải chuyển hướng sang liên hệ trực tiếp với cơ sở y tế thay vì tiếp tục cho thử vô hạn.
- Sau khi khôi phục thành công, phải gửi thông báo tới địa chỉ liên hệ khác đã lưu trước đó (nếu có) như một lớp cảnh báo bổ sung, đề phòng khôi phục bị người khác thực hiện.
- Toàn bộ dữ liệu dùng để xác minh danh tính trong quy trình khôi phục phải được xử lý tuân thủ quy định bảo mật dữ liệu y tế (không log thông tin xác minh dưới dạng plaintext lâu dài).
- Cho phép người giám hộ hợp pháp (ví dụ cha mẹ của bệnh nhân trẻ em) thực hiện khôi phục thay, với một quy trình xác minh vai trò giám hộ riêng biệt, có audit trail rõ ràng phân biệt "bệnh nhân tự khôi phục" và "giám hộ khôi phục thay".
