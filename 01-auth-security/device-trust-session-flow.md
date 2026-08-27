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

---

## Ngân hàng số quản lý device trust cho giao dịch chuyển tiền giá trị lớn

**Repository:** `device-trust-fintech-highvalue-transfer`

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền, mở tài khoản tiết kiệm, và thực hiện các giao dịch tài chính giá trị lớn.

**Vai trò của flow:** Thiết bị mới phải qua xác minh bổ sung trước khi được thực hiện giao dịch giá trị cao, dù đã đăng nhập thành công bằng mật khẩu/MFA thông thường — tách biệt rõ "trust để đăng nhập xem thông tin" và "trust để thực hiện giao dịch lớn".

**Yêu cầu cụ thể:**
- Thiết bị đăng nhập lần đầu chỉ được cấp trust ở mức xem thông tin cơ bản; muốn chuyển khoản vượt một ngưỡng giá trị phải trải qua một cửa sổ "device maturity" (ví dụ thiết bị phải hoạt động ổn định qua một khoảng thời gian, hoặc hoàn tất xác minh bổ sung) trước khi được nâng lên mức trust cho phép giao dịch lớn.
- Phải chặn được kịch bản kẻ tấn công vừa chiếm được thông tin đăng nhập, đăng nhập trên thiết bị mới rồi cố chuyển tiền lớn ngay trong vài giây — tức là race condition giữa việc thiết lập trust thiết bị mới và việc thực hiện giao dịch giá trị cao gần như tức thời sau đó phải được xử lý an toàn theo hướng mặc định từ chối.
- Cần phân biệt hợp lý giữa "thiết bị thực sự mới lạ" và "thiết bị cũ bị cài lại app/xóa dữ liệu" của chính chủ tài khoản, để tránh làm phiền người dùng hợp lệ mỗi lần họ cài lại app trong khi vẫn giữ được mức bảo vệ cần thiết cho trường hợp thật sự đáng ngờ.
- Khi giao dịch bị chặn vì thiết bị chưa đủ trust, phải có luồng step-up rõ ràng và tức thời (ví dụ xác minh qua kênh độc lập như gọi điện, video call, hoặc sinh trắc học bổ sung) cho các trường hợp hợp lệ nhưng cần gấp, thay vì bắt người dùng chờ hàng giờ/ngày một cách cứng nhắc.
- Mọi thay đổi trust level của thiết bị (nâng lên, hạ xuống, revoke) phải có audit trail không thể chỉnh sửa, và phải cảnh báo ngay qua kênh độc lập (SMS/email) mỗi khi có thiết bị mới đạt được mức trust cho phép giao dịch lớn, để chủ tài khoản phát hiện sớm nếu không phải do chính họ thực hiện.

---

## Nền tảng streaming giới hạn số thiết bị xem đồng thời trên một tài khoản

**Repository:** `device-trust-streaming-device-slot-limit`

**Hệ thống:** Một nền tảng streaming video trả phí, gói cước giới hạn số thiết bị được phép xem đồng thời (ví dụ tối đa N thiết bị/slot cùng lúc).

**Vai trò của flow:** Quản lý "device slot" đang chiếm quyền xem đồng thời, xử lý đúng khi thiết bị cũ không chủ động logout mà bị thiết bị mới thay thế, và hạn chế chia sẻ tài khoản vượt quá phạm vi cho phép của gói cước.

**Yêu cầu cụ thể:**
- Khi hai thiết bị mới cùng cố giành slot cuối cùng gần như đồng thời (ví dụ hai người trong nhà cùng bấm play khi chỉ còn một slot trống), hệ thống phải đảm bảo không cho cả hai cùng chiếm slot vượt giới hạn, đồng thời cơ chế khóa dùng để xử lý race condition này không được làm chậm đáng kể trải nghiệm bắt đầu phát video.
- Khi thiết bị cũ bị "đá" ra do hết slot vì thiết bị mới giành chỗ, phải dừng phát ngay trên thiết bị bị đá kèm thông báo rõ lý do trên màn hình, không để người dùng gặp phải một lỗi timeout khó hiểu không rõ nguyên nhân.
- Xác định một thiết bị đã "ngừng hoạt động" để tự động giải phóng slot là bài toán khó vì thường không có tín hiệu logout rõ ràng (app bị kill nền, mất mạng đột ngột, đóng tab trình duyệt) — cần cơ chế heartbeat/timeout hợp lý để tránh vừa giữ slot ảo cho thiết bị đã tắt thật sự, vừa tránh giải phóng nhầm slot của một thiết bị chỉ đang mất kết nối tạm thời.
- Cần phân biệt giữa việc cùng một tài khoản được xem trên nhiều thiết bị của một hộ gia đình hợp lệ, với dấu hiệu chia sẻ tài khoản vượt phạm vi cho phép (ví dụ các thiết bị đăng nhập từ nhiều thành phố/quốc gia khác nhau trong cùng khung giờ), để áp dụng chính sách slot chặt hơn cho trường hợp nghi ngờ mà không cản trở người dùng hợp lệ.
- Khi người dùng chủ động quản lý danh sách thiết bị đang chiếm slot và force-logout một thiết bị cụ thể, thao tác này phải giải phóng slot gần như ngay lập tức để thiết bị khác vào xem được, không để người dùng phải chờ hoặc thử lại nhiều lần.
