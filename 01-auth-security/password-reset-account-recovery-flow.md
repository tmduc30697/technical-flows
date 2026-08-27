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

---

## App di động xử lý khôi phục tài khoản khi số điện thoại đăng ký bị SIM-swap

**Repository:** `account-recovery-mobile-sim-swap`

**Hệ thống:** Một app di động dùng số điện thoại làm định danh chính và kênh nhận OTP để đăng nhập/khôi phục tài khoản.

**Vai trò của flow:** Xử lý rủi ro khi kẻ tấn công đã chiếm được quyền kiểm soát số điện thoại đăng ký (SIM-swap) và dùng chính OTP hợp lệ để chiếm tài khoản, thay vì mặc định coi việc nhận đúng OTP là bằng chứng danh tính tuyệt đối.

**Yêu cầu cụ thể:**
- Không được coi việc nhận đúng OTP qua SMS là yếu tố xác minh danh tính duy nhất/đủ mạnh cho các hành động nhạy cảm (ví dụ đổi email liên kết, đổi phương thức khôi phục) — các hành động này cần thêm một factor độc lập với số điện thoại, chẳng hạn thiết bị đã được tin cậy từ trước.
- Khi số điện thoại đăng ký vừa được đổi SIM (suy luận được từ tín hiệu nhà mạng nếu có, hoặc từ việc OTP đột ngột nhận trên một thiết bị hoàn toàn khác lịch sử trước đó), hệ thống nên áp dụng một khoảng cooling-off trước khi cho phép dùng số đó để thực hiện các thay đổi quan trọng trên tài khoản.
- Nếu tài khoản còn một thiết bị đã được tin cậy trước đó đang đăng nhập, các yêu cầu khôi phục hoặc đổi thông tin quan trọng đến từ số điện thoại vừa nhận OTP mới nên được đối chiếu và cảnh báo tới thiết bị tin cậy đó trước khi thực thi, để chủ tài khoản thật có cơ hội phát hiện và chặn kịp thời.
- Cần phân biệt giữa người dùng thật đổi điện thoại hợp lệ (mua máy mới, đổi SIM chính chủ) và kẻ tấn công SIM-swap — không thể chặn cứng mọi thay đổi đến từ số điện thoại mới vì sẽ khóa nhầm người dùng hợp lệ, nên cần kết hợp thêm tín hiệu khác như lịch sử thiết bị hoặc email dự phòng để quyết định mức độ tin cậy phù hợp.
- Khi phát hiện dấu hiệu SIM-swap sau khi đã xảy ra (chủ tài khoản thật báo cáo bị chiếm quyền), phải có luồng khôi phục ưu tiên không còn phụ thuộc vào số điện thoại đó nữa (ví dụ dựa vào email dự phòng đã xác minh từ trước hoặc xác minh danh tính thủ công), đồng thời khóa ngay các thay đổi mà kẻ tấn công đã thực hiện trong thời gian chiếm quyền.

---

## SaaS B2B khôi phục quyền truy cập khi công ty khách hàng đổi hoặc mất Identity Provider

**Repository:** `account-recovery-b2b-saas-sso-idp-lost`

**Hệ thống:** Một SaaS doanh nghiệp mà phần lớn tài khoản đăng nhập qua SSO của công ty khách hàng (Okta/Azure AD...), không có mật khẩu riêng trên nền tảng.

**Vai trò của flow:** Xử lý tình huống công ty khách hàng đổi nhà cung cấp IdP, mất quyền quản trị IdP cũ, hoặc IdP cũ ngừng hoạt động — khiến toàn bộ nhân viên công ty đó không thể đăng nhập qua SSO như bình thường, cần một quy trình khôi phục quyền truy cập không phụ thuộc vào chính IdP đã hỏng.

**Yêu cầu cụ thể:**
- Vì tài khoản SSO-only không có mật khẩu, khi IdP của cả một tổ chức khách hàng ngừng hoạt động, toàn bộ nhân viên công ty đó bị khóa cùng lúc — cần một cơ chế khôi phục cấp tổ chức, không phải xử lý từng user riêng lẻ, để tránh hàng trăm ticket hỗ trợ cùng lúc và tránh gián đoạn nghiệp vụ kéo dài.
- Việc xác minh ai có quyền yêu cầu chuyển đổi hoặc cấu hình lại SSO cho một tổ chức phải chặt chẽ, để tránh kẻ tấn công giả danh admin công ty khách hàng nhằm chiếm quyền truy cập toàn bộ dữ liệu tổ chức đó thông qua việc trỏ SSO sang một IdP do chúng kiểm soát.
- Cần thiết lập sẵn từ trước một tài khoản break-glass cấp tổ chức cho ít nhất một admin của mỗi tổ chức khách hàng lớn, không phải tạo mới lúc khẩn cấp, để có đường vào dự phòng khi SSO chính hỏng hoàn toàn, kèm quy trình giám sát chặt việc sử dụng tài khoản này.
- Khi công ty khách hàng chủ động đổi IdP theo kế hoạch (không phải sự cố), quy trình cutover phải cho phép chạy song song cả IdP cũ và mới trong một khoảng thời gian chuyển tiếp, để tránh có khoảnh khắc không IdP nào hoạt động khiến toàn bộ nhân viên bị khóa giữa chừng.
- Phải phân biệt rõ giữa lỗi tạm thời của IdP (downtime ngắn, sự cố mạng) và việc tổ chức thực sự cần chuyển IdP, tránh kích hoạt quy trình khôi phục khẩn cấp cấp tổ chức — vốn có rủi ro bảo mật cao hơn — chỉ vì một sự cố ngắn hạn có thể tự phục hồi.

---

## Nền tảng y tế cân bằng giữa khôi phục tài khoản nhanh cho tình huống cấp cứu và xác minh danh tính chặt

**Repository:** `account-recovery-healthcare-urgent-verification`

**Hệ thống:** Một nền tảng y tế cho bệnh nhân xem hồ sơ bệnh án, kết quả xét nghiệm, và đặt lịch khám; dữ liệu thuộc loại nhạy cảm cao và chịu quy định bảo vệ dữ liệu sức khỏe nghiêm ngặt.

**Vai trò của flow:** Xử lý mâu thuẫn giữa việc bệnh nhân có thể cần truy cập gấp (ví dụ đang ở phòng cấp cứu cần xem lịch sử dị ứng thuốc/bệnh nền) và yêu cầu xác minh danh tính đủ chặt để không cho người không liên quan xem hồ sơ sức khỏe của người khác.

**Yêu cầu cụ thể:**
- Cần phân tầng mức độ khẩn cấp của yêu cầu khôi phục: khôi phục thông thường (quên mật khẩu) đi qua luồng xác minh tiêu chuẩn, nhưng có một luồng khẩn cấp y tế riêng cho phép nhân viên y tế đang cấp cứu yêu cầu quyền xem tạm thời có kiểm soát mà không phải chờ hoàn tất toàn bộ bước xác minh danh tính đầy đủ của bệnh nhân.
- Luồng khẩn cấp phải giới hạn phạm vi truy cập được cấp, ví dụ chỉ xem dị ứng và thuốc đang dùng chứ không cấp toàn quyền chỉnh sửa hồ sơ hoặc xem thông tin tài chính, và tự động hết hạn sau một khoảng thời gian ngắn thay vì tồn tại như một quyền truy cập lâu dài.
- Mọi lần sử dụng luồng khẩn cấp phải được ghi log chi tiết và tự động kích hoạt review sau sự việc — ai đã cấp quyền, dựa trên căn cứ gì, đã xem gì — vì đây là lối tắt bỏ qua xác minh chuẩn nên cần cơ chế giám sát để phát hiện lạm dụng, không thể chỉ tin tưởng vào lý do "trường hợp khẩn cấp" mà không kiểm tra lại.
- Với khôi phục tài khoản thông thường không khẩn cấp, vì dữ liệu sức khỏe nhạy cảm hơn dữ liệu thông thường, mức xác minh danh tính phải cao hơn chuẩn phổ biến (không chỉ dựa vào email), chấp nhận đánh đổi tốc độ chậm hơn để giảm rủi ro lộ hồ sơ bệnh án cho người không phải chủ tài khoản.
- Phải xử lý được trường hợp người thân hoặc người giám hộ hợp pháp cần khôi phục quyền truy cập thay cho bệnh nhân không tự thao tác được, ví dụ bệnh nhân bất tỉnh, trẻ em, hoặc người cao tuổi — quy trình ủy quyền này cần xác minh mối quan hệ hợp pháp một cách có kiểm soát và tách biệt hẳn khỏi luồng tự khôi phục thông thường của chính chủ tài khoản.
