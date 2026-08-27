# Inventory reservation flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống có tồn kho hữu hạn (e-commerce đa kênh, marketplace nhiều seller, F&B đặt nguyên liệu, thời trang có size/màu, dropshipping) để luyện việc đặt trước/giữ/trả lại tồn kho chính xác dưới tải đồng thời cao.

---

## Giữ tồn kho khi thêm vào giỏ hàng trên sàn e-commerce nhiều kho

**Repository:** `inventory-reservation-ecommerce-multi-warehouse`

**Hệ thống:** Một sàn e-commerce có tồn kho phân bổ ở nhiều kho khác nhau theo khu vực, khách hàng thêm sản phẩm vào giỏ và tồn kho được "soft reserve" trong 15 phút khi vào bước checkout.

**Vai trò của flow:** Flow reservation phải chọn đúng kho gần khách để giữ hàng, trừ tồn kho tạm thời (không phải trừ thật) khi vào checkout, và giải phóng đúng nếu khách rời trang mà không thanh toán.

**Yêu cầu cụ thể:**
- Khi 2 khách ở 2 khu vực khác nhau cùng checkout sản phẩm chỉ còn 1 đơn vị tồn kho ở kho gần cả hai, dùng update nguyên tử `UPDATE inventory SET reserved = reserved + 1 WHERE available - reserved > 0` (không phải đọc số dư trước rồi update sau) để chỉ 1 trong 2 giữ được hàng, người thua nhận thông báo ngay và được đề xuất kho khác (nếu có) hoặc thông báo hết hàng.
- Reservation phải có TTL và bảng lưu trạng thái riêng (không chỉ tăng/giảm số nguyên trong cột `reserved`), để có thể truy ngược "ai đang giữ, giữ bao lâu, khi nào hết hạn" phục vụ debug và hiển thị đúng cho khách.
- Mô tả cụ thể race giữa job giải phóng reservation hết hạn và request thanh toán đến gần đồng thời: transaction thanh toán phải kiểm tra reservation vẫn còn hiệu lực (chưa bị job expire xóa) ngay trước khi trừ tồn kho thật, nếu đã bị expire phải từ chối thanh toán rõ ràng và không charge tiền.
- Khi khách sửa số lượng trong giỏ hàng (tăng từ 1 lên 3) trong lúc đang giữ reservation, phải tính lại đúng phần chênh lệch cần giữ thêm một cách atomic, không phải hủy reservation cũ rồi tạo mới (dễ mất tồn kho đã giữ trong khoảng thời gian giữa 2 bước).
- Có cơ chế đồng bộ giữa tồn kho "hiển thị công khai trên trang sản phẩm" và tồn kho "khả dụng thực" (available - reserved), quy định độ trễ chấp nhận được và tránh hiển thị "còn hàng" khi thực chất đã được reserve hết bởi các giỏ hàng khác.

---

## Giữ tồn kho khi seller và buyer thao tác đồng thời trên marketplace C2C

**Repository:** `inventory-reservation-c2c-marketplace`

**Hệ thống:** Một marketplace C2C (kiểu Facebook Marketplace) cho phép seller đăng nhiều sản phẩm với số lượng, buyer có thể đặt mua và seller cũng có thể tự sửa/xóa số lượng bất cứ lúc nào.

**Vai trò của flow:** Flow reservation phải xử lý được xung đột giữa hành động buyer (đặt mua, giữ tồn kho) và hành động seller (tự sửa số lượng, ngừng bán) xảy ra đồng thời trên cùng 1 sản phẩm.

**Yêu cầu cụ thể:**
- Khi seller sửa số lượng tồn kho từ 5 xuống 2 đúng lúc có 3 buyer khác nhau đang giữ reservation (tổng 3 đơn vị đã reserve), quy định rõ hành vi: các reservation đã tạo trước khi seller sửa vẫn hợp lệ (không tự hủy), chỉ số lượng còn lại để bán mới bị giới hạn theo giá trị mới — không để số liệu "âm" (reserved vượt tồn kho mới) gây lỗi hiển thị.
- Khi seller bấm "Ngừng bán" (đóng sản phẩm) đúng lúc buyer đang ở giữa bước xác nhận đặt mua đã giữ reservation trước đó, request xác nhận của buyer phải được xử lý dựa trên trạng thái tại thời điểm reservation được tạo (buyer đã giữ hợp lệ thì vẫn được mua), còn buyer mới sau khi sản phẩm đã đóng không thể tạo reservation mới — mô tả rõ ranh giới thời điểm nào được tính là "trước/sau".
- 2 buyer cùng bấm "Mua ngay" cho sản phẩm chỉ còn 1 số lượng gần như đồng thời: chỉ 1 người giữ được reservation, thiết kế bằng update nguyên tử trên cột tồn kho khả dụng, có test giả lập concurrency xác nhận không có 2 reservation cùng tồn tại cho 1 đơn vị hàng.
- Reservation của buyer phải có TTL ngắn hơn ở marketplace C2C so với e-commerce chính thức (vì tốc độ trao đổi/chat trước khi chốt mua có thể chậm), quy định giá trị TTL và cách gia hạn khi buyer và seller đang chat để thống nhất giao dịch.
- Có audit log cho mọi thay đổi tồn kho (ai sửa, khi nào, giá trị trước/sau) để xử lý tranh chấp khi buyer khiếu nại "đã đặt được hàng nhưng sau đó bị báo hết".

---

## Đặt trước hàng dropshipping khi tồn kho thực tế nằm ở nhà cung cấp bên thứ ba

**Repository:** `inventory-reservation-dropshipping-supplier`

**Hệ thống:** Một sàn dropshipping hiển thị tồn kho lấy từ API của nhiều nhà cung cấp khác nhau, tồn kho thực tế không do sàn kiểm soát trực tiếp mà chỉ đồng bộ định kỳ.

**Vai trò của flow:** Flow reservation phải giữ "chỗ" tạm thời ở tồn kho cache nội bộ khi khách đặt mua, đồng thời xác nhận thật với nhà cung cấp trước khi chốt đơn, xử lý đúng khi tồn kho cache bị lệch so với thực tế.

**Yêu cầu cụ thể:**
- Vì tồn kho từ nhà cung cấp chỉ đồng bộ mỗi vài phút (không real-time), yêu cầu khi khách đặt mua phải: (1) trừ tạm tồn kho cache nội bộ ngay (atomic, tránh 2 khách cùng đặt đơn vị cuối theo cache), (2) gọi API xác nhận đặt hàng thật với nhà cung cấp, (3) nếu nhà cung cấp báo hết hàng thật thì rollback lại cache và báo khách hủy đơn kèm hoàn tiền.
- Mô tả cụ thể race: 2 khách đặt mua sản phẩm mà cache nội bộ báo còn 1, cả 2 đều trừ cache thành công do cache đã lệch thực tế (nhà cung cấp chỉ còn 1 thật) — quy định thứ tự xử lý: request xác nhận với nhà cung cấp nào đến trước được ưu tiên giữ, request sau khi biết nhà cung cấp báo hết phải tự động hủy và hoàn tiền, có thông báo rõ cho khách thứ 2 nêu nguyên nhân (lệch dữ liệu tồn kho từ nhà cung cấp).
- Đồng bộ tồn kho từ nhà cung cấp về cache nội bộ phải là update atomic không được ghi đè mất các thay đổi tạm thời (reservation đang giữ) tạo ra bởi các đơn đang xử lý song song trong lúc đồng bộ diễn ra.
- Khi API nhà cung cấp timeout ở bước xác nhận đặt hàng thật, quy định retry tối đa bao nhiêu lần trong bao lâu trước khi coi là thất bại và hoàn tiền tự động cho khách — không để đơn hàng ở trạng thái "đang xử lý" quá lâu mà không có hành động rõ ràng.
- Có cơ chế cảnh báo khi tỷ lệ hủy đơn do lệch tồn kho với 1 nhà cung cấp cụ thể vượt ngưỡng (dấu hiệu đồng bộ dữ liệu với nhà cung cấp đó có vấn đề), để team vận hành có thể tạm ẩn sản phẩm của nhà cung cấp đó khỏi sàn.

---

## Trừ tồn nguyên liệu dùng chung nhiều món khi quán ăn nhận đơn đồng thời

**Repository:** `inventory-reservation-fnb-shared-ingredient`

**Hệ thống:** Một hệ thống order F&B cho nhà hàng/quán ăn, nhận đơn qua nhiều kênh (tại chỗ, app riêng, đối tác giao đồ ăn), mỗi món dùng nguyên liệu lấy từ kho bếp chung, và 1 nguyên liệu có thể xuất hiện trong công thức của nhiều món khác nhau.

**Vai trò của flow:** Flow reservation nguyên liệu phải trừ đúng số lượng theo công thức khi đơn được xác nhận, xử lý đúng khi nhiều đơn hàng từ các kênh khác nhau cùng cần chung 1 nguyên liệu sắp hết trong cùng thời điểm.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: đơn hàng 1 (từ app) gọi món "Phở bò" và đơn hàng 2 (khách tại chỗ) gọi món "Bún bò", cả 2 món cùng dùng nguyên liệu "thịt bò" mà kho chỉ còn đủ cho 1 trong 2 món, 2 đơn đến gần như đồng thời từ 2 kênh khác nhau — yêu cầu update nguyên tử trừ tồn nguyên liệu có điều kiện đủ số lượng, đơn nào trừ được trước thì món đó được xác nhận, đơn kia phải nhận báo "hết nguyên liệu, món tạm ngừng phục vụ" ngay lập tức, không phải sau khi đã in phiếu chế biến gửi xuống bếp.
- Với 1 đơn gồm nhiều món và mỗi món có thể chia sẻ nguyên liệu với món khác trong cùng đơn, yêu cầu trừ đồng thời tất cả nguyên liệu liên quan của toàn bộ các món trong 1 transaction atomic duy nhất khi xác nhận đơn (không xác nhận từng món riêng lẻ), để tránh trường hợp món đầu trừ được nguyên liệu thành công nhưng món sau trong cùng đơn phát hiện thiếu nguyên liệu khác giữa chừng, khiến đơn hàng rơi vào trạng thái nửa xác nhận nửa không.
- Khi tồn 1 nguyên liệu giảm xuống dưới ngưỡng cảnh báo (ví dụ chỉ đủ cho dưới 3 suất), yêu cầu các món dùng nguyên liệu đó phải được đánh dấu "sắp hết"/tạm ẩn trên menu hiển thị theo thời gian gần thực trên tất cả các kênh, tránh khách tiếp tục đặt món chắc chắn sẽ bị từ chối vì nguyên liệu vừa bị đơn khác đặt trước đó vài giây dùng hết.
- Khi 1 đơn bị hủy sau khi đã trừ nguyên liệu (khách đổi ý trước khi bếp bắt đầu chế biến), việc hoàn trả nguyên liệu về kho phải atomic và chỉ được thực hiện nếu đơn hủy trước mốc thời gian rõ ràng (ví dụ trước khi bếp xác nhận bắt đầu chế biến), tránh hoàn nhầm nguyên liệu đã thực sự được dùng để chế biến món.
- Với các kênh bán (app, tại chỗ, đối tác giao đồ ăn) cùng dùng chung 1 kho nguyên liệu vật lý, quy định độ trễ đồng bộ tồn kho nguyên liệu chấp nhận được giữa các kênh, và cơ chế chặn nhận đơn mới ngay khi 1 kênh phát hiện nguyên liệu vừa hết, để các kênh còn lại không tiếp tục nhận đơn dựa trên số liệu tồn kho đã lỗi thời.

---

## Đặt trước tồn kho theo biến thể size/màu khi tồn kho tổng vẫn hiển thị "còn hàng"

**Repository:** `inventory-reservation-fashion-size-color-variant`

**Hệ thống:** Một sàn thời trang, mỗi sản phẩm có nhiều biến thể (size S/M/L, nhiều màu sắc), mỗi biến thể có tồn kho riêng, trong khi trang sản phẩm chỉ hiển thị trạng thái "còn hàng" tổng hợp từ toàn bộ các biến thể.

**Vai trò của flow:** Flow reservation phải trừ đúng tồn kho ở cấp biến thể cụ thể mà khách chọn, xử lý đúng khi biến thể đó vừa hết ngay tại thời điểm khách bấm mua dù trang sản phẩm khách đang xem vẫn hiển thị "còn hàng".

**Yêu cầu cụ thể:**
- Mô tả cụ thể: sản phẩm áo có biến thể "size M, màu đen" chỉ còn đúng 1 đơn vị, 2 khách đang xem cùng trang sản phẩm (trang vẫn hiển thị "còn hàng" vì các biến thể khác còn nhiều) cùng chọn đúng size M màu đen và bấm mua gần như đồng thời — yêu cầu update nguyên tử theo khóa tổ hợp (product_id, size, color) chứ không phải theo product_id chung, để chỉ 1 khách giữ được đơn vị cuối, khách còn lại nhận báo hết đúng biến thể đó (không phải thông báo hết cả sản phẩm gây hiểu nhầm).
- Khi khách đổi biến thể trong lúc đang giữ reservation (ví dụ đổi từ size M sang size L ngay trên trang chi tiết trước khi vào bước thanh toán), thao tác đổi phải là 1 bước atomic gồm nhả reservation biến thể cũ và giữ reservation biến thể mới, không để khoảng hở giữa 2 thao tác khiến biến thể mới bị người khác giữ mất, hoặc biến thể cũ vẫn bị giữ dư thừa không cần thiết nếu bước giữ biến thể mới thất bại.
- Quy định rõ trạng thái "còn hàng" hiển thị tổng trên trang sản phẩm chỉ là gợi ý tổng quan (tổng các biến thể còn lớn hơn 0), không phải cam kết biến thể cụ thể còn hàng; tại bước khách chọn biến thể và bấm mua, hệ thống phải luôn truy vấn lại tồn kho khả dụng thực của đúng biến thể đó tại thời điểm request, không dựa vào trạng thái đã cache lúc khách load trang.
- Với sản phẩm có các biến thể phụ thuộc lẫn nhau khi hiển thị (ví dụ chọn màu trước rồi mới hiện các size còn hàng của màu đó), mô tả cụ thể race khi tồn kho biến thể thay đổi ngay trong lúc khách đang thao tác chọn tuần tự (màu rồi size) — giao diện và backend phải xử lý được trường hợp option vừa hiển thị "còn hàng" khi khách chọn màu nhưng đã hết ngay khi khách chọn xong size, trả lỗi rõ ràng ngay tại bước đó thay vì để khách đi hết qua bước thanh toán rồi mới báo lỗi.
- Nếu có kênh bán tại cửa hàng vật lý dùng chung tồn kho biến thể với kênh online (omni-channel), quy định cơ chế và độ trễ đồng bộ khi 1 biến thể vừa được bán hết tại cửa hàng vật lý, đảm bảo tồn kho online của đúng biến thể đó được cập nhật kịp thời để tránh nhận đơn online cho hàng thực tế đã không còn.
