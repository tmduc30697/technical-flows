# Waitlist management flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống có danh sách chờ (nhà hàng, khóa học online giới hạn học viên, chuyến bay overbooking, phòng gym/lớp học giới hạn chỗ, sản phẩm hết hàng chờ restock) để luyện việc quản lý thứ tự chờ, thăng cấp từ waitlist khi có chỗ trống, và xử lý đúng khi nhiều sự kiện xảy ra đồng thời ảnh hưởng đến danh sách chờ.

---

## Danh sách chờ bàn ăn nhà hàng theo thời gian thực

**Repository:** `waitlist-restaurant-realtime`

**Hệ thống:** Một app quản lý nhà hàng cho khách đăng ký vào danh sách chờ khi hết bàn, tự động gọi tên khách kế tiếp khi có bàn trống.

**Vai trò của flow:** Flow phải thăng cấp đúng khách kế tiếp trong danh sách chờ khi có bàn trống, xử lý đúng khi nhiều bàn trống ra gần như đồng thời (khách vừa ăn xong) và khi khách trong danh sách chờ tự hủy hoặc không phản hồi.

**Yêu cầu cụ thể:**
- Thứ tự trong danh sách chờ phải dựa trên timestamp đăng ký thực tế được ghi nhận atomic (dùng auto-increment hoặc timestamp có độ chính xác đủ cao kèm tie-breaker theo ID), không dựa vào thứ tự hiển thị trên UI có thể bị trễ do cache.
- Mô tả cụ thể: 2 bàn trống ra cùng lúc (2 nhóm khách thanh toán gần như đồng thời) và có 3 khách đang chờ theo thứ tự A, B, C — hệ thống phải gọi đúng A và B (2 người đầu hàng đợi) theo atomic increment quét danh sách, không để tình huống 2 nhân viên phục vụ ở 2 khu vực khác nhau cùng gọi trùng khách A cho 2 bàn khác nhau.
- Khi khách được gọi (thăng cấp khỏi waitlist) không đến trong khoảng thời gian quy định (ví dụ 5 phút), phải tự động chuyển sang gọi khách kế tiếp và đưa khách không đến vào cuối danh sách (hoặc loại khỏi danh sách theo chính sách) — xử lý atomic để không có 2 khách cùng được coi là "đang được gọi" cho cùng 1 bàn.
- Nếu khách tự hủy đăng ký chờ đúng lúc nhân viên đang thao tác gọi khách đó lên bàn trống, quy định rõ ai thắng dựa trên thời điểm transaction thực sự hoàn tất trước, tránh tình trạng gọi tên khách đã hủy hoặc mất bàn trống do xử lý sai thứ tự.
- Hiển thị vị trí hàng đợi cho khách đang chờ (qua app/SMS) phải phản ánh đúng số liệu real-time, cập nhật ngay khi có thay đổi (bàn trống ra, người phía trước hủy), không delay quá lâu gây trải nghiệm sai lệch.

---

## Danh sách chờ đăng ký khóa học online giới hạn học viên

**Repository:** `waitlist-online-course-enrollment`

**Hệ thống:** Một platform học online có khóa học giới hạn số lượng học viên (để đảm bảo chất lượng tương tác), khi đủ số lượng, học viên mới phải vào danh sách chờ và tự động được ghi danh khi có người rút khóa học.

**Vai trò của flow:** Flow phải thăng cấp đúng học viên kế tiếp trong danh sách chờ khi có slot trống, xử lý đúng khi nhiều học viên rút khóa học gần như đồng thời và nhiều học viên trong waitlist cùng cố gắng tự xác nhận giữ chỗ.

**Yêu cầu cụ thể:**
- Khi có slot trống (do có người rút), hệ thống phải tự động gửi lời mời cho đúng người đầu tiên trong danh sách chờ và giữ chỗ đó cho họ trong 1 khoảng thời gian (ví dụ 24 giờ) để hoàn tất thanh toán, dùng cơ chế hold có TTL tương tự flow booking, không mở slot cho toàn bộ danh sách chờ cùng tranh nhau.
- Mô tả cụ thể: 3 học viên rút khóa học gần như đồng thời (mở ra 3 slot), danh sách chờ có 5 người theo thứ tự — hệ thống phải mời đúng 3 người đầu tiên, mỗi lời mời độc lập với TTL riêng, và nếu 1 trong 3 không xác nhận kịp, chỉ đúng slot đó được chuyển cho người thứ 4, không ảnh hưởng tới 2 lời mời đang hiệu lực khác.
- Đảm bảo việc đếm "số slot đang trống thực" (đã trừ các slot đang giữ cho người được mời từ waitlist) là atomic, để tránh mời quá số slot thực có (ví dụ chỉ có 2 slot trống nhưng do tính sai lại mời tới 3 người).
- Khi học viên tự hủy đăng ký chờ đúng lúc hệ thống đang gửi lời mời cho họ (do vừa có slot trống), quy định rõ nếu hủy được ghi nhận trước khi lời mời được xác nhận gửi, học viên đó bị loại khỏi waitlist và slot chuyển ngay cho người kế tiếp, atomic không để lời mời "đi vào khoảng không".
- Nếu học viên vừa nhận lời mời từ waitlist vừa lúc đó lại tìm được khóa học khác và không muốn xác nhận nữa, họ có thể "nhường lại" slot ngay lập tức (không cần chờ hết TTL) — action này phải atomic chuyển ngay cho người kế tiếp trong danh sách, không để trống slot lãng phí thời gian chờ TTL hết hạn.

---

## Danh sách chờ overbooking cho chuyến bay khi có khách bị từ chối lên máy bay

**Repository:** `waitlist-airline-overbooking`

**Hệ thống:** Một hệ thống quản lý chuyến bay của hãng hàng không, do overbooking (bán vượt số ghế dự kiến 1 số khách no-show), khi tất cả khách đến đủ, hệ thống cần xử lý danh sách chờ standby để xếp khách vào ghế trống nếu có.

**Vai trò của flow:** Flow phải xử lý đúng khi có ghế trống xuất hiện gần giờ bay (do khách no-show hoặc hủy sát giờ) và xếp đúng khách trong danh sách standby theo đúng thứ tự ưu tiên (theo hạng thẻ thành viên, theo thời gian đăng ký chờ).

**Yêu cầu cụ thể:**
- Thứ tự ưu tiên trong danh sách standby phải được tính toán rõ ràng (ví dụ: hạng thẻ kim cương ưu tiên trước, cùng hạng thì theo thời gian đăng ký sớm hơn) và việc xếp ghế phải atomic theo đúng thứ tự này, không xử lý theo thứ tự request đến của nhân viên quầy (có thể không đúng ưu tiên thật).
- Mô tả cụ thể: 2 nhân viên ở 2 quầy check-in khác nhau cùng phát hiện có ghế trống (do 2 khách no-show riêng biệt được xác nhận gần như đồng thời) và cả 2 đều thao tác xếp khách từ danh sách standby — hệ thống phải đảm bảo mỗi ghế trống chỉ được gán 1 lần, dùng update atomic kiểm tra ghế còn trống trước khi gán, và cả 2 nhân viên đều phải xếp đúng người có ưu tiên cao nhất còn lại trong danh sách (không phải người đầu tiên nhân viên đó nhìn thấy).
- Mốc thời gian "đóng cửa check-in" (ví dụ 30 phút trước giờ bay) phải được áp dụng cứng cho việc xác định danh sách khách chưa check-in (candidate no-show), và ghế của khách chưa check-in chỉ được coi là "có thể trống" sau mốc này, không xử lý sớm hơn gây rủi ro xếp nhầm ghế cho khách sau đó vẫn đến kịp.
- Khi khách đang trong danh sách standby được gọi tên xếp ghế nhưng không có mặt tại cổng lên máy bay trong khoảng thời gian quy định, phải tự động chuyển ghế cho người kế tiếp trong danh sách ưu tiên, atomic để không giữ ghế trống vô ích tới giờ đóng cửa boarding.
- Ghi log đầy đủ toàn bộ quyết định xếp ghế standby (ai, ưu tiên gì, ghế nào, thời điểm) để phục vụ giải quyết khiếu nại khi khách cho rằng bị xếp sai thứ tự ưu tiên.

---

## Danh sách chờ lớp học gym/fitness có giới hạn số người mỗi buổi

**Repository:** `waitlist-gym-class-capacity`

**Hệ thống:** Một app đặt lớp học gym (yoga, HIIT) tại phòng tập, mỗi lớp giới hạn số lượng học viên, khi đủ chỗ học viên mới vào danh sách chờ.

**Vai trò của flow:** Flow phải thăng cấp học viên từ danh sách chờ khi có người trong lớp hủy đặt, xử lý đúng ngay cả khi việc hủy xảy ra rất gần giờ học (vài phút trước khi lớp bắt đầu).

**Yêu cầu cụ thể:**
- Khi có học viên hủy đặt chỗ, hệ thống phải kiểm tra danh sách chờ và tự động xác nhận chỗ cho người đầu tiên ngay lập tức nếu còn đủ thời gian trước giờ học (ví dụ trên 15 phút), atomic để không có 2 học viên trong danh sách chờ cùng được xác nhận cho 1 chỗ trống.
- Mô tả cụ thể: học viên A hủy đặt lớp đúng 20 phút trước giờ học, đồng thời học viên B (đang xếp hạng 1 trong waitlist) cũng đang tự thao tác thử đặt lại lớp đó ngay trong app (không biết là do waitlist) — quy định rõ 2 luồng (tự động thăng cấp từ waitlist và đặt trực tiếp thủ công) phải cùng đi qua 1 lớp kiểm tra chỗ trống thống nhất, tránh học viên B vô tình đặt được chỗ qua đường tắt trong khi hệ thống cũng đang cố gán chỗ đó cho B qua waitlist (kết quả đúng nhưng không được double-count).
- Nếu việc hủy xảy ra quá gần giờ học (dưới ngưỡng thời gian tối thiểu, ví dụ 15 phút), quy định rõ có thông báo cho danh sách chờ hay không (có thể không đủ thời gian cho học viên chờ kịp đến), và nếu có thông báo, phải mời nhiều người cùng lúc dạng "ai xác nhận trước được" chứ không chờ tuần tự từng người vì không còn nhiều thời gian.
- Đảm bảo số lượng chỗ trong lớp (đã đặt + đang giữ cho người waitlist được mời) không vượt giới hạn lớp tại bất kỳ thời điểm nào, kể cả trong trường hợp đặc biệt mời nhiều người cùng lúc ở tình huống sát giờ (chỉ người xác nhận đầu tiên được giữ, các lời mời khác tự động vô hiệu ngay khi 1 người xác nhận).
- Xử lý trường hợp phòng tập hạ số lượng giới hạn của lớp (ví dụ do sự cố thiết bị) sau khi đã có người trong danh sách chờ được thăng cấp — không hủy chỗ của người đã xác nhận, chỉ ảnh hưởng tới các slot chưa được lấp.

---

## Danh sách chờ nhận thông báo khi sản phẩm hết hàng được restock

**Repository:** `waitlist-product-restock-notify`

**Hệ thống:** Một sàn e-commerce cho phép khách đăng ký "báo tôi khi có hàng trở lại" cho sản phẩm hết hàng, khi restock sẽ gửi thông báo và cho phép mua theo thứ tự ưu tiên trong 1 khung giờ giới hạn.

**Vai trò của flow:** Flow phải gửi thông báo và mở quyền mua cho danh sách chờ theo đúng thứ tự khi có restock, xử lý đúng khi số lượng restock ít hơn số người trong danh sách chờ và nhiều người cùng cố mua ngay khi nhận thông báo.

**Yêu cầu cụ thể:**
- Khi restock, hệ thống phải gửi thông báo cho danh sách chờ theo lô/thứ tự ưu tiên (ví dụ người đăng ký sớm hơn được gửi thông báo trước vài phút, hoặc tất cả nhận cùng lúc nhưng có ưu tiên xử lý mua) — quy định rõ chính sách cụ thể được chọn và cách hệ thống thực thi đúng chính sách đó dưới tải cao khi nhiều người cùng bấm mua ngay khi nhận thông báo.
- Mô tả cụ thể: restock 50 sản phẩm, danh sách chờ có 500 người, tất cả nhận thông báo cùng lúc và cùng bấm mua trong vài giây đầu — việc trừ tồn kho khi mua phải atomic (giống flash sale) để chỉ đúng 50 người đầu tiên (theo thứ tự request xử lý thực tế tại DB, không theo thứ tự thông báo được gửi vì độ trễ gửi thông báo khác nhau) mua được, người còn lại nhận thông báo hết hàng ngay.
- Nếu chính sách ưu tiên người đăng ký sớm hơn: phải có cơ chế thực sự đảm bảo ưu tiên đó (ví dụ mở quyền mua theo lô thời gian, người đăng ký sớm nhận thông báo và cửa sổ mua trước vài phút so với người đăng ký sau), không chỉ gửi thông báo sớm hơn nhưng vẫn cho tất cả cùng bấm mua vào 1 thời điểm (khi đó ưu tiên thứ tự đăng ký không còn ý nghĩa thực tế).
- Sau khi hết hàng lại (do đã bán hết số lượng restock), những người còn lại trong danh sách chờ vẫn được giữ trong danh sách để chờ lần restock tiếp theo, không tự động xóa khỏi danh sách chỉ vì đã nhận thông báo lần này mà không mua được.
- Có giới hạn 1 người chỉ mua tối đa 1 đơn vị sản phẩm restock này (áp dụng atomic cùng với việc trừ tồn kho) để tránh 1 người mua hết nhiều đơn vị làm giảm số lượng người trong danh sách chờ thực sự nhận được hàng.
