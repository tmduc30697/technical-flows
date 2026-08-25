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

## Đặt trước nguyên liệu theo món ăn trong hệ thống đặt bàn nhà hàng

**Repository:** `inventory-reservation-restaurant-ingredient`

**Hệ thống:** Một nền tảng đặt bàn kết hợp đặt món trước (pre-order) cho nhà hàng, một số món đặc biệt có nguyên liệu giới hạn theo ngày (ví dụ chỉ có 20 phần bò Wagyu/ngày).

**Vai trò của flow:** Flow phải giữ trước số phần nguyên liệu giới hạn khi khách chọn món trong lúc đặt bàn, tránh việc nhận đặt món vượt số nguyên liệu bếp có thể chuẩn bị trong ngày.

**Yêu cầu cụ thể:**
- Khi nhiều khách đặt bàn khác nhau cùng chọn món Wagyu gần như đồng thời và tổng số phần yêu cầu vượt quá 20 phần còn lại, chỉ đúng số phần còn lại được xác nhận theo thứ tự request đến trước, các request sau nhận thông báo "món đã hết, chọn món khác" — yêu cầu cơ chế trừ tồn nguyên liệu atomic có kiểm tra ngưỡng, tương tự trừ tồn kho sản phẩm.
- Cho phép khách sửa đổi đơn đặt món trước giờ ăn (ví dụ đổi từ 2 phần Wagyu xuống 1 phần) phải trả lại đúng 1 phần vào pool chung ngay lập tức để phục vụ khách khác đang chờ, không giữ phần dư đó cho riêng đơn của khách đã sửa.
- Khi khách không đến (no-show) hoặc hủy đặt bàn sát giờ, quy định rõ nguyên liệu đã giữ có được tự động trả lại pool hay không tùy theo mốc thời gian hủy (ví dụ hủy trước 2 giờ mới được trả lại để bếp còn kịp dùng cho khách khác, hủy sát giờ thì không trả).
- Bếp cần biết chính xác số phần nguyên liệu đã được đặt trước tại mọi thời điểm (không chỉ số đơn hàng) để chuẩn bị đúng — cung cấp 1 nguồn dữ liệu duy nhất mà cả app khách và màn hình bếp đều đọc, tránh 2 nơi hiển thị số liệu lệch nhau do tính riêng.
- Xử lý trường hợp nhà hàng cập nhật giảm số lượng nguyên liệu giới hạn giữa ngày (ví dụ do nguồn cung thực tế ít hơn dự kiến) khi đã có một số đơn đặt trước — không được tự động hủy các đơn đã xác nhận, mà chỉ ngừng nhận đặt mới, và có cảnh báo nếu số đã xác nhận đã vượt số mới cập nhật.

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

## Giữ tồn kho theo size/màu trong flash sale thời trang

**Repository:** `inventory-reservation-fashion-flash-sale-variant`

**Hệ thống:** Một sàn thời trang online, mỗi sản phẩm có nhiều biến thể (size S/M/L, màu đen/trắng), tồn kho quản lý riêng theo từng biến thể, đang chạy sale giảm giá sâu.

**Vai trò của flow:** Flow reservation phải giữ đúng tồn kho theo biến thể cụ thể (không phải theo sản phẩm chung), tránh việc hết size M màu đen nhưng hệ thống báo hết cả sản phẩm hoặc ngược lại bán vượt tồn kho của riêng biến thể đó.

**Yêu cầu cụ thể:**
- Mỗi biến thể (SKU riêng theo size+màu) phải có dòng tồn kho riêng và được trừ/giữ độc lập, để việc trừ tồn kho size M màu đen không ảnh hưởng hoặc bị ảnh hưởng bởi trừ tồn kho size L màu trắng của cùng sản phẩm.
- Mô tả cụ thể: khách thêm vào giỏ 1 sản phẩm size M màu đen khi còn 2 đơn vị, giữ reservation, sau đó đổi ý chọn sang size L màu đen ngay trong giỏ hàng — yêu cầu hệ thống phải giải phóng đúng reservation cũ (size M) và tạo reservation mới (size L) trong 1 luồng atomic, không để lộ khoảng hở giữa 2 bước khiến người khác cướp được size M đang lẽ ra phải giữ tới lúc đổi xong.
- Khi 2 request đồng thời cùng cố giữ đơn vị tồn kho cuối cùng của 1 biến thể cụ thể (không phải cả sản phẩm), yêu cầu update nguyên tử theo đúng khóa biến thể (variant_id), có test concurrency riêng cho từng biến thể để đảm bảo lock không bị nhầm phạm vi (ví dụ lock nhầm theo product_id làm nghẽn không cần thiết giữa các biến thể khác nhau).
- Trang sản phẩm hiển thị tồn kho theo từng biến thể phải phản ánh đúng số liệu "khả dụng" (đã trừ phần đang được reserve bởi giỏ hàng người khác), quy định độ trễ tối đa cho phép giữa lúc 1 reservation được tạo/hết hạn và lúc trang sản phẩm cập nhật hiển thị.
- Xử lý trường hợp admin nhập thêm tồn kho cho 1 biến thể đang sắp hết (restock) đúng lúc có nhiều reservation đang hold gần hết TTL — số lượng mới nhập phải cộng đúng vào pool khả dụng ngay, không bị ghi đè mất bởi các transaction giữ/trả reservation đang chạy song song tại thời điểm restock.

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
