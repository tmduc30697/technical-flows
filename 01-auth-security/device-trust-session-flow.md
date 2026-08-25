# Device trust/session flow — Đề bài thực hành

Các đề bài dưới đây đi từ việc ghi nhớ thiết bị đáng tin để giảm ma sát đăng nhập, đến việc bắt buộc thiết bị phải đạt chuẩn bảo mật (Zero Trust) mới được truy cập, qua các bối cảnh fintech, B2B SaaS doanh nghiệp, streaming, messaging và e-commerce, luyện quản lý session đa thiết bị, phát hiện bất thường và revoke quyền truy cập.

---

## Ngân hàng số với tính năng "thiết bị đáng tin" để giảm số lần hỏi MFA

**Repository:** `device-trust-digital-bank-trusted-device`

**Hệ thống:** Một app ngân hàng số, người dùng đăng nhập bằng password + MFA.

**Vai trò của flow:** Sau khi xác thực MFA thành công một lần, thiết bị được đánh dấu "đáng tin" trong một khoảng thời gian, các lần đăng nhập tiếp theo từ thiết bị đó chỉ cần password, không phải nhập lại MFA — nhưng vẫn phải giữ được khả năng thu hồi trust từ xa.

**Yêu cầu cụ thể:**
- Trust của thiết bị phải gắn với một device fingerprint đủ mạnh (kết hợp nhiều tín hiệu: thiết bị, browser/app version, không chỉ riêng cookie dễ bị xóa/giả mạo), và phải tự hết hạn sau một khoảng thời gian cố định (ví dụ 30 ngày) dù không có hoạt động bất thường.
- Người dùng phải có màn hình "Thiết bị đã tin cậy" để xem danh sách và revoke trust của từng thiết bị riêng lẻ, revoke phải có hiệu lực ngay ở lần request kế tiếp từ thiết bị đó (không chờ token hết hạn tự nhiên).
- Khi phát hiện tín hiệu bất thường từ một thiết bị đã được trust trước đó (vị trí địa lý nhảy bất hợp lý, thay đổi network subnet đáng ngờ), hệ thống phải tự động yêu cầu MFA lại dù trust chưa hết hạn theo thời gian.
- Đổi mật khẩu hoặc phát hiện tài khoản bị compromise phải tự động revoke trust của toàn bộ thiết bị đã lưu, buộc mọi thiết bị (kể cả đã trust) phải MFA lại ở lần đăng nhập kế tiếp.
- Test được race condition: request đăng nhập và request revoke trust của cùng thiết bị xảy ra gần như đồng thời — không được để trạng thái cuối cùng phụ thuộc vào thứ tự xử lý không xác định.

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

## Nền tảng streaming giới hạn số thiết bị đồng thời

**Repository:** `device-trust-streaming-device-limit`

**Hệ thống:** Một dịch vụ xem phim/nghe nhạc trực tuyến trả phí, gói cước cơ bản giới hạn số thiết bị được xem đồng thời.

**Vai trò của flow:** Quản lý session theo thiết bị để thực thi giới hạn concurrent-device theo gói cước, và cho người dùng tự quản lý thiết bị đang đăng nhập.

**Yêu cầu cụ thể:**
- Khi người dùng đăng nhập từ thiết bị mới vượt quá giới hạn gói cước, hệ thống phải có chính sách rõ ràng (ví dụ tự đăng xuất thiết bị lâu không hoạt động nhất, hoặc hỏi người dùng chọn thiết bị muốn đăng xuất) — không được để vượt giới hạn một cách âm thầm.
- Phân biệt được "một thiết bị đang phát" và "một thiết bị đã đăng nhập nhưng không phát" — chính sách giới hạn có thể áp dụng khác nhau cho hai trạng thái này (ví dụ giới hạn concurrent stream chặt hơn giới hạn số thiết bị đăng nhập).
- Người dùng phải có màn hình "Quản lý thiết bị" để xem danh sách thiết bị đang đăng nhập (loại thiết bị, lần hoạt động cuối, vị trí ước lượng) và tự đăng xuất từng thiết bị.
- Khi phát hiện một tài khoản bị dùng đồng thời từ nhiều vị trí địa lý cách xa nhau bất hợp lý trong thời gian ngắn (dấu hiệu chia sẻ tài khoản/lộ mật khẩu), phải có cơ chế cảnh báo hoặc yêu cầu xác thực lại, tách biệt với logic giới hạn thiết bị theo gói cước.
- Downgrade gói cước (từ gói nhiều thiết bị xuống gói ít thiết bị) phải áp dụng giới hạn mới ngay từ lần đăng nhập kế tiếp, xử lý rõ trường hợp số thiết bị hiện tại đang vượt giới hạn mới.

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

## Bảng điều khiển admin nội bộ yêu cầu tái xác thực thiết bị giữa phiên làm việc

**Repository:** `device-trust-internal-admin-reauth`

**Hệ thống:** Một bảng điều khiển quản trị nội bộ cho phép nhân viên vận hành thao tác trực tiếp trên hệ thống production (xem log nhạy cảm, thay đổi cấu hình).

**Vai trò của flow:** Session không chỉ được cấp một lần khi đăng nhập mà phải được tái xác minh định kỳ dựa trên posture thiết bị hiện tại, đảm bảo một session dài hạn không bị lợi dụng khi trạng thái thiết bị thay đổi giữa lúc dùng.

**Yêu cầu cụ thể:**
- Ngoài việc kiểm tra posture khi bắt đầu session, hệ thống phải định kỳ (ví dụ mỗi 10-15 phút) kiểm tra lại trạng thái thiết bị hiện tại (mã hóa ổ đĩa, antivirus, phiên bản OS) trong suốt thời gian session còn hoạt động.
- Nếu phát hiện posture xấu đi giữa phiên (ví dụ tắt mã hóa ổ đĩa), phải buộc đăng xuất ngay hành động đang thực hiện dở, không chờ tới lần kiểm tra định kỳ kế tiếp nếu tín hiệu đủ nghiêm trọng.
- Các hành động có mức rủi ro cao (xóa dữ liệu, thay đổi quyền truy cập của người khác) phải yêu cầu một bước xác nhận tái xác thực riêng dù session posture-check vẫn đang pass, tách biệt khỏi kiểm tra posture định kỳ.
- Session phải có thời hạn tối đa tuyệt đối (ví dụ 8 giờ) buộc đăng nhập lại từ đầu dù mọi kiểm tra posture định kỳ đều pass liên tục, tránh session "sống mãi" nhờ gia hạn liên tục.
- Ghi log chi tiết mọi lần posture-check thất bại dẫn đến buộc đăng xuất (thiết bị nào, lý do gì, đang thực hiện hành động gì) để phục vụ điều tra sự cố an ninh sau này.

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
