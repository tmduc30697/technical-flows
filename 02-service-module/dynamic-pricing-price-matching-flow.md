# Dynamic pricing/price matching flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (đặt xe, e-commerce, đặt phòng khách sạn, giao đồ ăn, quảng cáo real-time bidding) để luyện việc tính giá động theo thời gian thực và đối soát/khớp giá một cách nhất quán, minh bạch.

---

## Surge pricing cho ứng dụng đặt xe

**Repository:** `dynamic-pricing-ride-hailing-surge`

**Hệ thống:** Một ứng dụng đặt xe điều chỉnh giá cước theo thời gian thực dựa trên tỷ lệ cung/cầu (số xe trống so với số yêu cầu đặt xe) tại một khu vực.

**Vai trò của flow:** Dynamic pricing dùng để cân bằng cung cầu tức thời — tăng giá khi thiếu xe để khuyến khích thêm tài xế hoạt động và giảm nhu cầu, nhưng phải minh bạch và công bằng với người dùng.

**Yêu cầu cụ thể:**
- Giá surge phải được tính lại liên tục theo khu vực địa lý nhỏ (không tính chung cho toàn thành phố) dựa trên dữ liệu cung/cầu gần thời gian thực, và giá hiển thị cho khách phải được chốt (lock) ngay khi họ xem giá ước tính, không thay đổi giữa lúc xem và lúc xác nhận đặt xe trong một khoảng thời gian ngắn hợp lý.
- Xử lý trường hợp khách xác nhận đặt xe ngay lúc giá đang chuyển từ mức surge cao xuống thấp (hoặc ngược lại) — phải rõ ràng khách trả đúng giá đã hiển thị lúc xác nhận, không bị tính lại theo giá mới sau khi đơn đã được tạo.
- Đặt ngưỡng trần cho hệ số surge tối đa (ví dụ không quá 3 lần giá gốc) để tránh giá tăng vô hạn trong tình huống cực đoan (thiên tai, sự kiện lớn) gây phản ứng tiêu cực từ người dùng và rủi ro pháp lý.
- Đảm bảo hệ thống tính surge không bị lợi dụng bởi hành vi giả tạo cung/cầu (ví dụ nhiều tài xế đồng thời tắt app trong một khu vực để đẩy giá lên rồi mở lại) — cần cơ chế phát hiện biến động cung bất thường không tương ứng với biến động nhu cầu thực tế.
- Ghi log đầy đủ hệ số surge áp dụng cho mỗi chuyến đi (giá trị, thời điểm, khu vực) để phục vụ giải thích minh bạch khi khách hàng khiếu nại về giá, và phục vụ phân tích sau này về hiệu quả của chính sách surge.

---

## Đối chiếu và khớp giá đối thủ cho sàn e-commerce (price matching)

**Repository:** `dynamic-pricing-ecommerce-price-matching`

**Hệ thống:** Một sàn e-commerce có chính sách "khớp giá đối thủ" — nếu khách hàng tìm thấy giá thấp hơn ở nơi khác cho cùng sản phẩm, hệ thống tự động hoặc theo yêu cầu sẽ điều chỉnh giá bằng giá đó.

**Vai trò của flow:** Flow này xử lý việc thu thập giá đối thủ, xác thực yêu cầu khớp giá của khách hàng, và áp dụng điều chỉnh giá một cách có kiểm soát để không bị lợi dụng.

**Yêu cầu cụ thể:**
- Khi khách hàng gửi yêu cầu khớp giá kèm link/ảnh chứng minh giá đối thủ, hệ thống phải có quy trình xác thực (tự động qua crawler kiểm tra giá thực tế còn hiệu lực, hoặc review thủ công) trước khi áp dụng, tránh khớp giá theo thông tin giả hoặc đã lỗi thời.
- Định nghĩa rõ điều kiện sản phẩm được coi là "giống nhau" để khớp giá (cùng SKU/mã sản phẩm, cùng tình trạng mới/cũ, cùng chính sách bảo hành) — tránh khớp giá nhầm giữa hai sản phẩm chỉ giống tên nhưng khác thông số.
- Xử lý trường hợp giá đối thủ thay đổi hoặc hết hàng ngay sau khi yêu cầu khớp giá được gửi nhưng trước khi được xử lý xong — phải có cơ chế kiểm tra lại tại thời điểm áp dụng, không áp dụng giá đã không còn tồn tại trên thực tế.
- Đặt giới hạn hợp lý cho việc khớp giá (ví dụ không áp dụng cho sản phẩm đang giảm giá flash sale, giới hạn số lần một tài khoản được yêu cầu khớp giá trong một khoảng thời gian) để tránh lạm dụng chính sách gây thiệt hại biên lợi nhuận.
- Đảm bảo giá đã khớp chỉ áp dụng cho đúng đơn hàng/khách hàng yêu cầu (không tự động thay đổi giá công khai cho tất cả khách hàng khác), trừ khi chính sách quy định rõ là điều chỉnh giá toàn sàn.

---

## Định giá động theo nhu cầu và thời điểm đặt cho khách sạn/vé máy bay

**Repository:** `dynamic-pricing-travel-demand-based`

**Hệ thống:** Một nền tảng đặt phòng khách sạn tự điều chỉnh giá theo tỷ lệ phòng còn trống, thời điểm đặt so với ngày lưu trú, và mùa cao/thấp điểm.

**Vai trò của flow:** Dynamic pricing ở đây vận hành theo chu kỳ chậm hơn surge pricing (thay đổi theo giờ/ngày, không phải theo giây), nhưng cần đảm bảo tính nhất quán qua nhiều bước của quy trình đặt phòng dài hơn.

**Yêu cầu cụ thể:**
- Giá hiển thị trong kết quả tìm kiếm chỉ là tham khảo tại thời điểm tìm kiếm; giá thực tế phải được xác nhận lại (re-quote) ngay trước bước thanh toán cuối cùng, và nếu giá đã tăng, phải yêu cầu khách hàng xác nhận lại rõ ràng thay vì tự động tính giá mới mà không thông báo.
- Khi khách hàng đã đặt và thanh toán một phòng với một mức giá, các thay đổi giá động sau đó (do nhu cầu tăng) không được ảnh hưởng tới booking đã xác nhận — giá đã chốt phải cố định trong toàn bộ vòng đời của booking đó.
- Xử lý trường hợp hai khách hàng cùng xem một phòng cuối cùng của một loại phòng cùng lúc với hai mức giá khác nhau (do refresh trang ở hai thời điểm khác nhau) — người xác nhận đặt trước (win race condition) phải nhận đúng giá của mình, không có tình huống double-booking phòng đã hết.
- Đảm bảo thuật toán định giá không tạo ra chênh lệch giá bất hợp lý giữa các khách hàng khác nhau xem cùng một phòng trong cùng một khung thời gian ngắn (tránh cảm giác bị phân biệt giá thiếu minh bạch), trừ khi có chính sách giá theo phân khúc khách hàng được công bố rõ ràng.
- Cung cấp cho đối tác khách sạn (đối tượng cung cấp phòng) khả năng xem lại lịch sử giá đã áp dụng theo từng ngày để đối soát doanh thu, vì giá động ảnh hưởng trực tiếp tới phần chia sẻ doanh thu với đối tác.

---

## Phí giao hàng động theo khoảng cách, thời tiết và nhu cầu cho app giao đồ ăn

**Repository:** `dynamic-pricing-food-delivery-fee`

**Hệ thống:** Một app giao đồ ăn tính phí giao hàng động dựa trên khoảng cách, số lượng đơn hàng đang chờ giao trong khu vực, và điều kiện thời tiết.

**Vai trò của flow:** Dynamic pricing ở đây ảnh hưởng tới ba bên (khách hàng trả phí, tài xế nhận thù lao, nhà hàng không bị ảnh hưởng) nên cần tách bạch rõ ràng cách tính cho mỗi bên.

**Yêu cầu cụ thể:**
- Phí giao hàng hiển thị cho khách hàng lúc đặt đơn phải được chốt tại thời điểm đặt và không thay đổi dù điều kiện (thời tiết, số đơn chờ) biến động trong lúc đơn đang được xử lý/giao.
- Thù lao động cho tài xế (có thể khác công thức so với phí khách trả, ví dụ nền tảng bù thêm khi thời tiết xấu để giữ chân tài xế) phải được tính và hiển thị minh bạch cho tài xế trước khi họ nhận đơn, không thay đổi sau khi đã nhận.
- Xử lý trường hợp một đơn hàng bị nhiều tài xế từ chối liên tục (do phí quá thấp so với khoảng cách/điều kiện) — hệ thống phải có cơ chế tự động tăng thù lao theo từng lần bị từ chối để tăng khả năng được nhận, tránh đơn hàng bị "treo" không ai nhận.
- Đảm bảo công thức tính phí không bị ảnh hưởng bất hợp lý bởi dữ liệu nhiễu (ví dụ một tài xế cố tình báo cáo vị trí sai để tạo cảm giác thiếu tài xế cục bộ) — cần đối chiếu nhiều tín hiệu độc lập trước khi điều chỉnh phí.
- Cung cấp báo cáo minh bạch cho khách hàng về breakdown phí (phí cơ bản, phí khoảng cách, phụ phí thời tiết/nhu cầu) ngay tại màn hình xác nhận đơn, tránh cảm giác phí "bí ẩn" gây mất niềm tin.

---

## Định giá real-time bidding cho nền tảng quảng cáo

**Repository:** `dynamic-pricing-adtech-real-time-bidding`

**Hệ thống:** Một ad exchange nơi các nhà quảng cáo đấu giá theo thời gian thực (dưới 100ms) để hiển thị quảng cáo cho một lượt truy cập trang web cụ thể.

**Vai trò của flow:** Dynamic pricing ở đây là việc xác định giá thắng thầu (clearing price) cho mỗi lượt hiển thị quảng cáo, đòi hỏi độ chính xác và tốc độ xử lý cực cao, khác hẳn về độ trễ so với các bối cảnh trên.

**Yêu cầu cụ thể:**
- Toàn bộ vòng đấu giá (nhận yêu cầu, gửi tới các nhà quảng cáo, nhận giá thầu, xác định người thắng) phải hoàn tất trong ngân sách thời gian rất chặt (ví dụ dưới 100ms) — thiết kế phải ưu tiên timeout nghiêm ngặt cho từng nhà quảng cáo tham gia, loại bỏ giá thầu tới muộn dù có thể là giá cao nhất.
- Áp dụng đúng cơ chế đấu giá đã chọn (ví dụ second-price auction: người thắng trả giá của người trả giá cao thứ hai, không phải giá mình đã trả) và phải kiểm chứng được logic này bằng test với nhiều bộ giá thầu khác nhau.
- Xử lý trường hợp một nhà quảng cáo gửi giá thầu bất thường (quá cao so với lịch sử, có dấu hiệu lỗi hệ thống phía họ) — có ngưỡng hợp lý để loại bỏ giá thầu ngoại lai (outlier) tránh ảnh hưởng tới tính công bằng của phiên đấu giá.
- Đảm bảo tính nhất quán khi có hàng nghìn phiên đấu giá chạy đồng thời trên nhiều lượt hiển thị khác nhau — mỗi phiên phải độc lập hoàn toàn, không để trạng thái/giá thầu của một phiên ảnh hưởng nhầm sang phiên khác do lỗi chia sẻ state.
- Ghi log đầy đủ và có thể kiểm toán được giá thắng thầu, danh sách người tham gia, và giá từng người trả cho mỗi phiên đấu giá, phục vụ đối soát doanh thu với nhà quảng cáo và giải quyết tranh chấp khi có nghi ngờ gian lận đấu giá.
