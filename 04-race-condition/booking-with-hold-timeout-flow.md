# Booking with hold & timeout flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống đặt chỗ (khách sạn, vé máy bay, rạp phim, phòng họp, sân thể thao) để luyện việc giữ chỗ tạm thời (hold) có thời hạn, xử lý hết hạn/hủy/xác nhận đồng thời mà không double-book hoặc kẹt tồn kho.

---

## Giữ ghế vé máy bay trong lúc khách điền thông tin thanh toán

**Repository:** `booking-hold-airline-seat-payment`

**Hệ thống:** Một nền tảng đặt vé máy bay, khách chọn ghế và có 10 phút để hoàn tất thanh toán trước khi ghế được thả lại cho người khác.

**Vai trò của flow:** Flow tạo một "hold" trên ghế đã chọn ngay khi khách bắt đầu checkout, tự động giải phóng nếu hết 10 phút mà chưa thanh toán xong, và phải loại trừ được việc 2 khách cùng giữ 1 ghế.

**Yêu cầu cụ thể:**
- Khi 2 khách cùng bấm chọn ghế 12A trong cùng 1 giây, chỉ đúng 1 người tạo hold thành công; yêu cầu dùng constraint unique hoặc `SELECT ... FOR UPDATE` trên dòng ghế, người thua phải nhận lỗi rõ ràng "ghế đã được giữ" ngay lập tức, không phải chờ timeout mới biết.
- Hold phải có `expires_at` lưu trong DB (không chỉ dựa vào TTL cache), và có background job quét hold hết hạn để giải phóng ghế — quy định chu kỳ quét (ví dụ mỗi 30 giây) và xử lý trường hợp thanh toán vừa xong đúng lúc job đang giải phóng ghế đó (race giữa "confirm" và "expire").
- Khi khách thanh toán thành công ở giây thứ 599 (gần hết hold 600 giây) đúng lúc background job expire đang chạy, transaction confirm phải thắng nếu hold chưa bị xóa tại thời điểm confirm bắt đầu — dùng optimistic check (so `expires_at` hoặc version) để đảm bảo không xác nhận vé trên 1 hold đã bị giải phóng.
- Cho phép khách gia hạn hold thêm 1 lần (ví dụ +5 phút) nếu đang ở màn hình nhập thẻ, nhưng phải giới hạn số lần gia hạn để tránh 1 người giữ ghế vô hạn định chặn người khác.
- Khi hủy hold thủ công (khách bấm "Hủy") đúng lúc job expire tự động cũng đang xử lý cùng hold đó, đảm bảo ghế chỉ được giải phóng đúng 1 lần, không có lỗi double-release hoặc giải phóng nhầm ghế khác.

---

## Giữ phòng khách sạn qua kênh OTA (Booking.com-like) với đồng bộ tồn phòng đa kênh

**Repository:** `booking-hold-hotel-ota-multi-channel`

**Hệ thống:** Một hệ thống quản lý khách sạn (channel manager) đồng bộ tồn phòng cho nhiều kênh bán (website riêng, OTA A, OTA B) cùng lúc.

**Vai trò của flow:** Khi khách bắt đầu đặt phòng ở bất kỳ kênh nào, flow phải hold phòng đó trên toàn bộ các kênh khác trong một khoảng thời gian ngắn, tránh việc 2 kênh cùng bán 1 phòng cuối cùng.

**Yêu cầu cụ thể:**
- Khi request hold đến từ kênh A cho phòng cuối cùng của 1 loại phòng/ngày, phải cập nhật ngay tồn kho chung (giảm về 0) trước khi trả response cho kênh A, để request gần như đồng thời từ kênh B đọc thấy tồn kho đã hết — không cho phép cả 2 kênh đọc tồn kho "còn 1" cùng lúc rồi cả 2 đều hold thành công.
- Hold từ mỗi kênh phải có TTL riêng theo đặc thù kênh (ví dụ OTA cho khách 15 phút để nhập thẻ, website riêng có thể ngắn hơn), và khi hết hạn phải trả tồn kho lại cho pool chung để tất cả kênh đều thấy phòng trống trở lại.
- Mô tả cụ thể race condition: hold của kênh A hết hạn đúng lúc kênh B đang gọi API kiểm tra tồn kho để hiển thị "còn phòng" cho khách — đảm bảo job giải phóng hold và cập nhật tồn kho là 1 transaction atomic, không có khoảng hở khiến kênh B đọc dữ liệu tồn kho cũ.
- Khi mạng đến 1 kênh OTA chậm/timeout ngay sau khi hold đã tạo thành công ở DB nội bộ nhưng response chưa kịp gửi cho OTA đó, quy định cách xử lý: hold vẫn phải tồn tại và tự expire theo TTL bình thường (không rollback chỉ vì response bị timeout ở lớp network), tránh mất đồng bộ giữa DB nội bộ và trạng thái phía OTA.
- Có API cho phép admin khách sạn hủy hold thủ công (ví dụ khách gọi điện hủy), phải đảm bảo hủy thủ công và job tự động expire không đụng nhau gây lỗi giải phóng 2 lần hoặc giải phóng nhầm hold mới hơn đã được tạo lại cho phòng đó.

---

## Đặt suất khám bệnh online với hold slot trong hệ thống bệnh viện

**Repository:** `booking-hold-hospital-appointment-slot`

**Hệ thống:** Một app đặt lịch khám bệnh cho phòng khám, mỗi bác sĩ có các slot giờ cố định, bệnh nhân chọn slot và có 5 phút để xác nhận thông tin bảo hiểm trước khi slot được thả.

**Vai trò của flow:** Flow giữ slot tạm thời khi bệnh nhân chọn giờ khám, tự giải phóng nếu bệnh nhân không hoàn tất xác nhận, và đảm bảo không có 2 bệnh nhân cùng được xác nhận cho cùng 1 slot của cùng 1 bác sĩ.

**Yêu cầu cụ thể:**
- Mỗi slot (bác sĩ + giờ) chỉ có đúng 1 chỗ; khi 2 bệnh nhân bấm chọn cùng slot gần như đồng thời, dùng unique constraint trên (doctor_id, slot_time) ở bảng hold để DB tự chặn request thứ 2, không dựa vào kiểm tra "SELECT rồi INSERT" ở tầng application (dễ bị race).
- Khi hold hết hạn (bệnh nhân không xác nhận bảo hiểm trong 5 phút), slot phải hiển thị lại "còn trống" cho các bệnh nhân khác đang xem lịch gần như ngay lập tức — quy định độ trễ tối đa cho phép giữa lúc hold expire và lúc slot hiển thị trống trở lại (ví dụ dưới 10 giây).
- Cho phép lễ tân (staff) đặt slot thủ công qua hệ thống nội bộ cùng lúc bệnh nhân đang tự đặt online cho slot đó — 2 luồng khác nhau (app bệnh nhân, hệ thống nội bộ) phải cùng đi qua 1 lớp hold thống nhất, không có "đường tắt" nào bỏ qua kiểm tra hold.
- Nếu bệnh nhân xác nhận bảo hiểm thành công nhưng ngay lúc đó hệ thống bảo hiểm bên thứ ba timeout, quy định: hold phải được gia hạn thêm một khoảng ngắn để retry gọi bảo hiểm (ví dụ 1 lần, +2 phút), không để bệnh nhân mất slot chỉ vì lỗi tạm thời của bên thứ ba.
- Viết rõ hành vi khi bệnh nhân bấm "Hủy" giữa lúc network đang gửi request xác nhận bảo hiểm (request xác nhận và request hủy gần như đồng thời) — chỉ 1 trong 2 được áp dụng theo nguyên tắc rõ ràng (ví dụ dùng version/optimistic lock trên hold để request đến sau bị từ chối nếu trạng thái đã đổi).

---

## Đặt sân thể thao theo giờ với chính sách hold khác nhau cho khách vãng lai và member

**Repository:** `booking-hold-sports-court-membership`

**Hệ thống:** Một app đặt sân cầu lông/pickleball theo giờ, có 2 loại khách: member (đặt trước, hold lâu hơn) và khách vãng lai (đặt tại chỗ, hold ngắn).

**Vai trò của flow:** Flow phải xử lý 2 chính sách hold khác nhau trên cùng 1 tài nguyên (khung giờ sân), đảm bảo khách vãng lai không thể "cướp" slot đang được member hold hợp lệ, nhưng slot hold của khách vãng lai phải nhả rất nhanh nếu không thanh toán.

**Yêu cầu cụ thể:**
- Định nghĩa rõ 2 loại hold có TTL khác nhau (ví dụ member: 15 phút, khách vãng lai: 3 phút) nhưng cùng chia sẻ 1 tài nguyên slot — thiết kế bảng hold sao cho việc kiểm tra "slot còn trống" luôn nhất quán bất kể loại hold nào đang giữ.
- Mô tả race condition cụ thể: hold của khách vãng lai (TTL 3 phút) hết hạn đúng lúc member đang bấm xác nhận đặt slot đó — đảm bảo transaction xác nhận của member phải kiểm tra hold cũ đã thực sự được giải phóng (không còn active) trước khi tạo hold mới, tránh 2 hold chồng lên cùng 1 slot.
- Khi hệ thống thanh toán tại chỗ (khách vãng lai quẹt thẻ tại quầy) và thanh toán online (member qua app) cùng cố gắng xác nhận cho cùng 1 slot trong cửa sổ vài giây trước khi hold hết hạn, chỉ 1 giao dịch được xác nhận — mô tả cơ chế lock cụ thể (ví dụ advisory lock theo slot_id) để 2 luồng thanh toán khác kênh vẫn loại trừ nhau đúng.
- Cho phép nhân viên quầy huỷ hold của khách vãng lai sớm hơn TTL (ví dụ khách đổi ý ngay tại quầy) mà không ảnh hưởng tới các hold khác của slot liền kề, tránh lỗi huỷ nhầm slot do thao tác nhanh tay của nhân viên.
- Quy định hành vi khi cùng 1 member mở 2 tab/2 thiết bị và bấm giữ cùng 1 slot gần như đồng thời (tự đụng chính mình) — hệ thống phải coi đây là 1 hold duy nhất hoặc từ chối tab thứ 2 rõ ràng, không tạo 2 hold trùng cho cùng 1 user.

---

## Giữ vé xem phim theo ghế trong giờ vàng, có bán qua cả app và quầy vé tại rạp

**Repository:** `booking-hold-cinema-seat-omnichannel`

**Hệ thống:** Một hệ thống bán vé rạp phim, ghế được bán song song qua app di động và quầy vé tại rạp cho cùng 1 buổi chiếu.

**Vai trò của flow:** Flow hold ghế phải đồng bộ real-time giữa 2 kênh bán (app và quầy), timeout ngắn (2-3 phút) để không giữ ghế vô ích trong giờ vàng có nhu cầu cao.

**Yêu cầu cụ thể:**
- Khi khách ở quầy chọn ghế H5 trên màn hình bán vé của nhân viên đúng lúc 1 khách khác chọn ghế H5 trên app, chỉ 1 bên hold thành công — thiết kế để cả 2 kênh dùng chung 1 nguồn sự thật (DB trung tâm) cho trạng thái ghế, không cache riêng ở từng kênh mà không đồng bộ.
- TTL hold ở quầy nên ngắn hơn app (nhân viên xác nhận nhanh hơn khách tự thao tác) — nêu rõ giá trị TTL cụ thể cho từng kênh và lý do khác biệt.
- Mô tả cụ thể tình huống: khách trên app hold ghế H5, sau 2 phút 59 giây (gần hết hold) mới bấm thanh toán, đúng lúc job expire chạy ở giây 180 — yêu cầu transaction thanh toán phải re-validate hold còn hiệu lực ngay trước khi confirm (không tin vào trạng thái đã đọc từ trước), nếu hold đã hết hạn phải báo lỗi để khách chọn ghế khác ngay, không charge tiền rồi mới phát hiện ghế mất.
- Khi rạp bán hết ghế cho buổi chiếu (tất cả ghế đã confirm hoặc đang hold), cả app và quầy phải cùng lúc hiển thị "hết ghế"/"sold out" trong độ trễ tối đa cho phép — quy định cơ chế thông báo real-time (ví dụ qua sự kiện cập nhật đẩy tới cache hiển thị) thay vì polling chậm.
- Xử lý trường hợp mất kết nối tạm thời giữa quầy vé và server trung tâm (mạng rạp chập): quy định app vẫn phải hoạt động đúng, còn quầy vé phải chặn bán vé mới cho tới khi kết nối lại, tránh bán ghế đã hold/đã bán ở app trong lúc quầy bị mất kết nối.
