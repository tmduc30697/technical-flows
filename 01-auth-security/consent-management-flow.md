# Consent management flow (GDPR/privacy) — Đề bài thực hành

Các đề bài dưới đây đi từ vai trò một website/app đơn lẻ thu thập và quản lý sự đồng ý của người dùng (data controller), đến vai trò tự xây hạ tầng chia sẻ tín hiệu đồng ý cho hàng trăm đối tác (Consent Management Platform), qua các bối cảnh e-commerce, ad-tech, y tế, B2B SaaS và mobile app.

---

## E-commerce EU với banner cookie/tracking và trung tâm quản lý tùy chọn

**Repository:** `consent-ecommerce-cookie-banner`

**Hệ thống:** Một website thương mại điện tử bán hàng cho khách EU, dùng nhiều script bên thứ ba (analytics, quảng cáo, chat hỗ trợ).

**Vai trò của flow:** Thu thập và lưu trữ sự đồng ý của người dùng cho từng nhóm mục đích (cookie cần thiết, analytics, marketing) trước khi cho phép các script tương ứng chạy, theo yêu cầu GDPR/ePrivacy.

**Yêu cầu cụ thể:**
- Script phân tích/marketing/quảng cáo tuyệt đối không được nạp hoặc chạy trước khi người dùng đưa ra đồng ý cho đúng nhóm mục đích tương ứng — không dùng kiểu "chặn hình ảnh nhưng script vẫn chạy ngầm".
- Banner phải cho phép từ chối dễ như chấp nhận (không thiết kế nút "Chấp nhận tất cả" nổi bật còn "Từ chối"/"Tùy chỉnh" bị ẩn khó bấm).
- Lưu lại lịch sử đồng ý của mỗi người dùng (thời điểm, phiên bản chính sách, các mục đã chọn) để làm bằng chứng khi có yêu cầu kiểm tra từ cơ quan quản lý.
- Cho phép người dùng vào lại "Trung tâm quản lý quyền riêng tư" bất kỳ lúc nào để xem và thay đổi lựa chọn đã đồng ý trước đó, và việc rút lại đồng ý phải có hiệu lực ngay (tắt script liên quan) không cần load lại toàn trang.
- Khi chính sách quyền riêng tư có thay đổi ảnh hưởng tới phạm vi dữ liệu thu thập, phải yêu cầu người dùng đồng ý lại (re-consent) thay vì coi đồng ý cũ vẫn còn hiệu lực.

---

## Ad-tech platform tự xây hạ tầng Consent Management Platform (CMP) cho hàng trăm đối tác

**Repository:** `consent-adtech-cmp-platform`

**Hệ thống:** Một nền tảng ad-tech trung gian giữa publisher (website hiển thị quảng cáo) và hàng trăm đối tác quảng cáo/DSP cần biết trạng thái đồng ý của người dùng để quyết định có hiển thị quảng cáo cá nhân hóa hay không.

**Vai trò của flow:** Nền tảng đóng vai trò provider hạ tầng — phát sinh, lưu trữ và truyền phát tín hiệu đồng ý theo chuẩn liên ngành (kiểu IAB TCF) tới toàn bộ đối tác trong thời gian thực.

**Yêu cầu cụ thể:**
- Tín hiệu đồng ý phải được mã hóa/encode theo một định dạng chuẩn hóa (consent string) mà mọi đối tác downstream có thể decode và hiểu nhất quán, không phụ thuộc vào implementation riêng của từng publisher.
- Hệ thống phải xử lý được việc một publisher tích hợp nhiều đối tác quảng cáo khác nhau, mỗi đối tác có thể yêu cầu các mục đích (purposes) xử lý dữ liệu khác nhau — tín hiệu phải phản ánh đúng mục đích nào được đồng ý cho đối tác nào.
- Khi người dùng rút lại đồng ý, thay đổi phải được lan truyền tới toàn bộ đối tác đang tích hợp trong một khoảng thời gian đủ ngắn được cam kết (SLA), không để đối tác tiếp tục xử lý dữ liệu dựa trên tín hiệu đồng ý đã lỗi thời.
- Có cơ chế versioning cho consent string để khi chuẩn ngành cập nhật phiên bản mới, hệ thống vẫn đọc/ghi tương thích được với các đối tác chưa nâng cấp.
- Ghi log đầy đủ và có thể tra soát được: với một user cụ thể tại một thời điểm cụ thể, đối tác nào đã nhận tín hiệu gì — phục vụ việc điều tra khi có khiếu nại hoặc audit từ cơ quan quản lý.

---

## Nền tảng nghiên cứu y tế với đồng ý chia sẻ dữ liệu bệnh nhân cho đối tác nghiên cứu

**Repository:** `consent-healthcare-research-dynamic-consent`

**Hệ thống:** Một nền tảng kết nối bệnh viện với các đơn vị nghiên cứu y khoa, cần sự đồng ý rõ ràng của bệnh nhân trước khi chia sẻ dữ liệu (đã ẩn danh hoặc giả danh hóa) cho mục đích nghiên cứu.

**Vai trò của flow:** Quản lý đồng ý ở mức chi tiết theo từng mục đích nghiên cứu cụ thể, hỗ trợ "dynamic consent" — bệnh nhân có thể đồng ý/rút lại theo từng nghiên cứu riêng, không phải một lần đồng ý áp dụng cho mọi mục đích tương lai.

**Yêu cầu cụ thể:**
- Mỗi yêu cầu đồng ý phải gắn với một mục đích nghiên cứu cụ thể (tên nghiên cứu, phạm vi dữ liệu, đơn vị thực hiện) — không dùng một checkbox chung "đồng ý dùng dữ liệu cho nghiên cứu" mơ hồ.
- Khi có nghiên cứu mới muốn dùng lại dữ liệu đã thu thập trước đó cho mục đích khác với mục đích ban đầu, hệ thống phải yêu cầu xin đồng ý mới riêng cho mục đích đó, không tự động mở rộng phạm vi đồng ý cũ.
- Bệnh nhân phải rút được đồng ý cho một nghiên cứu cụ thể bất kỳ lúc nào, và hệ thống phải phân biệt được dữ liệu đã xử lý trước khi rút (có thể giữ theo quy định) và dữ liệu tương lai (phải ngừng chia sẻ ngay).
- Lưu trữ đầy đủ lịch sử phiên bản của từng bản đồng ý (nội dung đúng như bệnh nhân đã đọc tại thời điểm đồng ý), vì nội dung mô tả nghiên cứu có thể thay đổi theo thời gian và cần đối chiếu đúng phiên bản khi có tranh chấp.
- Với bệnh nhân là trẻ em hoặc người không đủ năng lực hành vi, luồng phải chuyển sang lấy đồng ý từ người giám hộ hợp pháp, có xác minh vai trò giám hộ riêng và audit trail phân biệt rõ.

---

## B2B SaaS với đồng ý theo tính năng và bằng chứng đồng ý cho mục đích thanh tra

**Repository:** `consent-b2b-saas-audit-trail`

**Hệ thống:** Một SaaS marketing automation cho doanh nghiệp khách hàng ở EU, xử lý dữ liệu liên hệ (email, số điện thoại) của khách hàng cuối của các doanh nghiệp đó.

**Vai trò của flow:** Quản lý đồng ý ở hai lớp — đồng ý của doanh nghiệp khách hàng với SaaS (data processing agreement) và đồng ý của khách hàng cuối với việc nhận email marketing — đồng thời cung cấp bằng chứng đồng ý khi doanh nghiệp khách hàng bị thanh tra.

**Yêu cầu cụ thể:**
- Hệ thống phải lưu trữ được nguồn gốc và thời điểm mỗi khách hàng cuối đưa ra đồng ý nhận marketing (double opt-in email, form trên web, import từ CRM có sẵn đồng ý...) để doanh nghiệp khách hàng có thể xuất bằng chứng khi bị khiếu nại.
- Khi khách hàng cuối bấm "unsubscribe"/rút đồng ý ở một chiến dịch, việc rút đồng ý phải áp dụng ngay cho toàn bộ chiến dịch tương lai của doanh nghiệp đó, không chỉ chiến dịch hiện tại.
- Phân biệt rõ giữa dữ liệu được xử lý theo "lợi ích hợp pháp" (không cần đồng ý riêng, ví dụ giao dịch đã mua hàng) và dữ liệu cần đồng ý rõ ràng (marketing quảng cáo) — không gộp chung một loại consent cho cả hai.
- Cho phép doanh nghiệp khách hàng chủ động export toàn bộ nhật ký đồng ý của khách hàng cuối của họ (định dạng có thể đọc được) để tự nộp cho cơ quan quản lý khi cần, không phải yêu cầu SaaS hỗ trợ thủ công mỗi lần.
- Khi một doanh nghiệp khách hàng ngừng sử dụng SaaS, phải có quy trình rõ ràng cho việc dữ liệu đồng ý liên quan được xóa/chuyển giao theo đúng data processing agreement đã ký, không giữ lại vô thời hạn.

---

## Mobile app với SDK bên thứ ba chỉ khởi tạo sau khi có đồng ý

**Repository:** `consent-mobile-sdk-lazy-init`

**Hệ thống:** Một mobile app tiêu dùng phổ thông tích hợp nhiều SDK bên thứ ba (analytics, quảng cáo, crash reporting), phát hành trên cả iOS và Android.

**Vai trò của flow:** Đảm bảo các SDK theo dõi/quảng cáo chỉ được khởi tạo và bắt đầu gửi dữ liệu sau khi người dùng đưa ra đồng ý rõ ràng, đồng thời tuân thủ các yêu cầu riêng của từng nền tảng (App Tracking Transparency trên iOS).

**Yêu cầu cụ thể:**
- Toàn bộ SDK có khả năng theo dõi (tracking) phải được thiết kế "lazy init" — không được để SDK tự động khởi tạo khi app mở lần đầu trước khi màn hình xin đồng ý xuất hiện.
- Trên iOS, luồng đồng ý nội bộ của app phải phối hợp đúng với App Tracking Transparency prompt của hệ điều hành — không tự suy ra "đã đồng ý" nếu người dùng chưa tương tác với prompt của OS.
- Người dùng phải vào được màn hình cài đặt quyền riêng tư trong app để xem và tắt riêng từng SDK/mục đích đã đồng ý trước đó, thay đổi phải có hiệu lực ngay cho các lần khởi động app tiếp theo.
- Khi người dùng từ chối tracking, app vẫn phải hoạt động đầy đủ chức năng chính (không được ép buộc đồng ý bằng cách chặn tính năng không liên quan tới tracking).
- Đồng bộ trạng thái đồng ý giữa các thiết bị của cùng một người dùng (nếu có đăng nhập tài khoản) — đồng ý/rút đồng ý trên một thiết bị phải phản ánh đúng khi họ dùng app trên thiết bị khác.

---

## Mạng xã hội xử lý re-consent khi chính sách quyền riêng tư thay đổi

**Repository:** `consent-social-policy-re-consent`

**Hệ thống:** Một mạng xã hội quy mô lớn định kỳ cập nhật chính sách quyền riêng tư (thêm đối tác dữ liệu mới, thay đổi mục đích xử lý dữ liệu).

**Vai trò của flow:** Quản lý việc yêu cầu người dùng đồng ý lại khi chính sách thay đổi vượt một ngưỡng "thay đổi trọng yếu", và duy trì lịch sử phiên bản đồng ý của từng người dùng qua nhiều lần cập nhật chính sách.

**Yêu cầu cụ thể:**
- Hệ thống phải phân loại được thay đổi chính sách nào là "trọng yếu" (cần re-consent bắt buộc) và thay đổi nào chỉ cần thông báo (không chặn truy cập) — không coi mọi cập nhật văn bản là như nhau.
- Với thay đổi trọng yếu, người dùng chưa đồng ý lại phải bị hạn chế ở một mức độ hợp lý (ví dụ chỉ chặn tính năng liên quan tới phần chính sách thay đổi) thay vì khóa hoàn toàn tài khoản một cách đột ngột.
- Lưu vết đầy đủ: mỗi người dùng đã đồng ý phiên bản chính sách nào, vào lúc nào, qua kênh nào (web/app) — có thể truy vấn lại lịch sử đầy đủ của một tài khoản cụ thể khi cần.
- Xử lý được quy mô lớn: việc yêu cầu re-consent phải rollout dần (không dồn tất cả người dùng cùng lúc gây quá tải hệ thống) mà vẫn đảm bảo mọi người dùng cuối cùng đều được yêu cầu trong một hạn định rõ ràng.
- Người dùng không hoạt động lâu ngày (dormant account) quay lại sau nhiều lần chính sách đã đổi phải được yêu cầu đồng ý lại đúng phiên bản chính sách mới nhất, không bị bỏ qua vì hệ thống chỉ kiểm tra "đã từng đồng ý một lần" chung chung.
