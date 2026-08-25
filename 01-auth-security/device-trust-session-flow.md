# Device trust/session flow — Đề bài thực hành

Các đề bài dưới đây đi từ việc ghi nhớ thiết bị đáng tin để giảm ma sát đăng nhập, đến việc bắt buộc thiết bị phải đạt chuẩn bảo mật (Zero Trust) mới được truy cập, qua các bối cảnh fintech, B2B SaaS doanh nghiệp, streaming, messaging và e-commerce, luyện quản lý session đa thiết bị, phát hiện bất thường và revoke quyền truy cập.

---

## B2B SaaS Zero Trust chỉ cho phép truy cập từ thiết bị công ty quản lý

**Repository:** `device-trust-zero-trust-b2b-saas`

**Hệ thống:** Một công cụ nội bộ doanh nghiệp chứa dữ liệu nhạy cảm (tài chính, nhân sự), công ty áp dụng chính sách Zero Trust: chỉ thiết bị đã được MDM (Mobile Device Management) quản lý và đạt chuẩn bảo mật mới được truy cập.

**Vai trò của flow:** Session chỉ được cấp khi thiết bị chứng minh được "posture" hợp lệ (certificate do MDM cấp, trạng thái mã hóa ổ đĩa, phiên bản OS đủ mới) — không dựa vào chỉ đăng nhập đúng mật khẩu.

**Yêu cầu cụ thể:**
- Request truy cập phải kèm theo chứng chỉ thiết bị (device certificate) do hệ thống MDM cấp, và server phải verify được certificate này còn hợp lệ (chưa bị revoke, đúng chuỗi tin cậy) trước khi cấp session.
- Nếu thiết bị không có certificate hợp lệ (BYOD, thiết bị cá nhân), phải từ chối truy cập với thông báo rõ ràng hướng dẫn liên hệ IT để đăng ký thiết bị, không hiển thị lỗi mơ hồ.
- Trong lúc session đang hoạt động, nếu MDM báo cáo thiết bị chuyển sang trạng thái không đạt chuẩn (ví dụ tắt mã hóa ổ đĩa, jailbreak/root), hệ thống phải buộc kết thúc session hiện tại gần như ngay lập tức, không chờ session tự hết hạn.
- Chính sách posture yêu cầu (phiên bản OS tối thiểu, bắt buộc mã hóa...) phải cấu hình được theo nhóm người dùng/phòng ban khác nhau, không phải một chính sách cứng áp dụng chung cho mọi người.
- Có luồng dự phòng có kiểm soát (ví dụ VPN tạm thời + xác minh thủ công bởi IT) cho trường hợp nhân viên cần truy cập gấp từ thiết bị không đạt chuẩn, được ghi log đầy đủ và giới hạn thời gian.

---

## App messaging với phiên đăng nhập liên kết đa thiết bị (linked devices)

**Repository:** `device-trust-messaging-linked-devices`

**Hệ thống:** Một app tin nhắn (giống WhatsApp/Telegram) cho phép dùng đồng thời trên điện thoại (thiết bị chính) và mở thêm trên web/desktop (thiết bị liên kết).

**Vai trò của flow:** Quản lý session dạng "thiết bị chính + nhiều thiết bị liên kết", đồng bộ trạng thái và cho phép revoke từng thiết bị liên kết mà không ảnh hưởng thiết bị chính.

**Yêu cầu cụ thể:**
- Việc liên kết thiết bị mới (web/desktop) phải được xác nhận trực tiếp từ thiết bị chính (ví dụ quét QR code hoặc approve trên điện thoại), không cho liên kết chỉ bằng thông tin đăng nhập từ xa.
- Mỗi thiết bị liên kết phải có khóa mã hóa session riêng, revoke một thiết bị liên kết không được ảnh hưởng tới session của thiết bị chính hoặc các thiết bị liên kết khác.
- Người dùng phải xem được danh sách đầy đủ thiết bị liên kết đang hoạt động (kèm thời điểm liên kết, lần hoạt động cuối) ngay trên thiết bị chính, và revoke được từng thiết bị riêng lẻ ngay lập tức.
- Nếu thiết bị chính bị mất/đăng xuất, phải có chính sách rõ ràng cho các thiết bị liên kết đang hoạt động (ví dụ tự động hết hạn sau một khoảng thời gian, hoặc yêu cầu liên kết lại với thiết bị chính mới).
- Đồng bộ tin nhắn/trạng thái đọc giữa các thiết bị liên kết phải xử lý được race condition khi hai thiết bị cùng thao tác gần như đồng thời (ví dụ đánh dấu đã đọc trên cả hai thiết bị cùng lúc) mà không gây trạng thái không nhất quán.

---

## Marketplace e-commerce phát hiện session bất thường và yêu cầu step-up

**Repository:** `device-trust-marketplace-anomaly-stepup`

**Hệ thống:** Một sàn thương mại điện tử cho phép người dùng lưu thông tin thanh toán và địa chỉ để mua hàng nhanh.

**Vai trò của flow:** Quản lý session "remember me" thông thường cho trải nghiệm mua sắm hằng ngày, nhưng phát hiện được dấu hiệu session bị chiếm quyền (session hijacking) để yêu cầu xác thực bổ sung đúng lúc mà không làm phiền người dùng hợp lệ.

**Yêu cầu cụ thể:**
- Session "remember me" (giữ đăng nhập lâu dài) phải được phân tách với các hành động nhạy cảm (thanh toán, đổi địa chỉ giao hàng, đổi phương thức thanh toán) — những hành động này yêu cầu xác thực bổ sung dù session chính vẫn còn hạn.
- Hệ thống phải theo dõi tín hiệu bất thường trong phiên (thay đổi IP/vị trí địa lý giữa các request liên tiếp một cách bất hợp lý, thay đổi user-agent giữa chừng) để gắn cờ session nghi ngờ bị đánh cắp.
- Khi một session bị gắn cờ nghi ngờ, hệ thống phải yêu cầu xác thực lại trước khi cho tiếp tục các hành động nhạy cảm, đồng thời không tự động đăng xuất ngay lập tức nếu chỉ đang xem sản phẩm (tránh làm phiền người dùng hợp lệ vì false positive).
- Khi phát hiện thanh toán được thực hiện ngay sau một thay đổi thông tin nhạy cảm gần đây (ví dụ vừa đổi địa chỉ giao hàng/phương thức thanh toán), phải tự động yêu cầu xác thực lại trước khi cho hoàn tất giao dịch, để chặn kịch bản chiếm session rồi đổi địa chỉ nhận hàng.
- Cung cấp cho người dùng lịch sử các lần đăng nhập/session gần đây (thiết bị, vị trí ước lượng, thời gian) và cách báo cáo nếu phát hiện session không phải của họ, kèm hành động tự động khóa tạm giao dịch đang treo khi người dùng báo cáo bị chiếm session.
