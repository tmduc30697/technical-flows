# Ride-hailing matching flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống điều phối theo thời gian thực (gọi xe cá nhân, giao đồ ăn, xe tải chở hàng, xe cấp cứu, đi chung xe) để luyện việc ghép nối tài xế/đơn hàng với khách trong điều kiện nhiều bên cùng cạnh tranh một nguồn lực di động và có vị trí thay đổi liên tục.

---

## Ghép tài xế cho khách gọi xe cá nhân trong khu vực đông đúc

**Repository:** `ride-hailing-driver-matching-dense-area`

**Hệ thống:** Một app gọi xe (kiểu Grab/Uber) tìm tài xế gần nhất cho khách vừa đặt xe, khu vực trung tâm giờ cao điểm có nhiều khách đặt và nhiều tài xế trống cùng lúc.

**Vai trò của flow:** Flow matching phải gán đúng 1 tài xế cho 1 khách, tránh việc 1 tài xế nhận được nhiều lời mời chuyến cùng lúc từ các khách khác nhau và vô tình nhận nhầm 2 chuyến, hoặc 1 khách bị gán trùng 2 tài xế.

**Yêu cầu cụ thể:**
- Khi hệ thống chọn tài xế gần nhất để gửi lời mời, phải đánh dấu tài xế đó ở trạng thái "đang được mời" (locked) ngay lập tức và atomic, để không có 2 khách khác nhau cùng được gợi ý gửi lời mời tới cùng 1 tài xế trong lúc đang chờ tài xế đó phản hồi.
- Mô tả cụ thể: tài xế nhận được lời mời từ khách A, đúng lúc đó khách B (đặt xe sau nhưng ở gần tài xế hơn) cũng được matching engine chọn trúng tài xế này do độ trễ cập nhật vị trí — yêu cầu request "mời tài xế" phải qua update atomic kiểm tra trạng thái tài xế (chỉ mời nếu đang "trống", chuyển ngay sang "đang được mời") để B không thể mời trùng tài xế đang xử lý lời mời của A.
- Tài xế chỉ có 1 khoảng thời gian ngắn để chấp nhận lời mời (ví dụ 15 giây); nếu hết thời gian mà không phản hồi, phải tự động trả tài xế về trạng thái "trống" và chuyển lời mời cho tài xế tiếp theo trong danh sách gần nhất — mô tả rõ cơ chế timeout này chạy độc lập và không xung đột nếu tài xế phản hồi "chấp nhận" đúng ngay tại thời điểm timeout kích hoạt (ai đến trước theo transaction thắng).
- Khi tài xế bấm "Chấp nhận" cho lời mời của khách A đúng lúc khách A đã hủy chuyến (do đợi quá lâu) gần như đồng thời, quy định rõ thứ tự xử lý: nếu request hủy của khách được ghi nhận (transaction hoàn tất) trước khi request chấp nhận của tài xế bắt đầu, chuyến bị hủy và tài xế nhận thông báo "khách đã hủy", không tạo chuyến đi "ma".
- Viết test giả lập 50 khách đặt xe cùng lúc trong khu vực có 10 tài xế trống, xác nhận không có tài xế nào bị gán cho 2 chuyến cùng lúc, và không có khách nào nhận được xác nhận từ 2 tài xế khác nhau.

---

## Ghép đơn giao đồ ăn cho shipper khi 1 shipper có thể nhận nhiều đơn cùng lúc (batch delivery)

**Repository:** `ride-hailing-food-delivery-batch-matching`

**Hệ thống:** Một app giao đồ ăn cho phép shipper nhận nhiều đơn cùng lúc nếu các quán ăn/địa điểm giao gần nhau (batch order), tăng hiệu quả giao hàng.

**Vai trò của flow:** Flow matching phải quyết định gộp đơn nào vào cùng 1 shipper một cách atomic, tránh việc 2 hệ thống matching (hoặc 2 lần chạy matching engine) gán trùng đơn cho nhiều shipper hoặc vượt quá số đơn tối đa 1 shipper được nhận cùng lúc.

**Yêu cầu cụ thể:**
- Mỗi đơn hàng chỉ được gán cho đúng 1 shipper tại 1 thời điểm — dùng update atomic đổi trạng thái đơn từ "chờ ghép" sang "đã gán cho shipper X" có kiểm tra điều kiện, để nếu matching engine chạy 2 lần gần nhau (do worker chạy song song hoặc retry) không gán trùng đơn cho 2 shipper khác nhau.
- Mô tả cụ thể: matching engine đang cố gán đơn thứ 4 vào batch của shipper đã có 3 đơn (giới hạn tối đa là 4), đúng lúc đơn thứ 3 của shipper đó bị khách hủy — quy định rõ hệ thống phải re-check số đơn hiện tại của shipper ngay tại thời điểm gán đơn mới (không dựa vào số liệu đã tính trước khi có sự kiện hủy), để không vượt giới hạn hoặc gán nhầm dựa trên dữ liệu cũ.
- Khi shipper đang trong batch 3 đơn và có 1 đơn mới ở gần được đề xuất gộp thêm, nhưng đúng lúc đó shipper bấm "Bắt đầu giao" (khóa batch, không nhận thêm đơn), request gộp đơn mới phải bị từ chối atomic nếu batch đã bị khóa, không cho phép thêm đơn vào 1 batch đã bắt đầu giao.
- Nếu 2 quán ăn hoàn tất chuẩn bị đơn hàng gần như đồng thời và cả 2 đơn đó đang được đề xuất gộp cho cùng 1 shipper, đảm bảo việc "xác nhận gộp" là atomic theo đúng thứ tự request đến, không để trạng thái batch bị ghi đè sai (ví dụ đơn 2 ghi đè mất thông tin đơn 1 do update không đúng cách, chỉ cộng thêm không thay thế).
- Có cơ chế phát hiện và cảnh báo khi tỷ lệ đơn bị "gán rồi hủy do re-check thất bại" tăng cao bất thường trong 1 khu vực, dấu hiệu matching engine đang chạy quá thường xuyên gây tranh chấp lẫn nhau.

---

## Điều phối xe tải chở hàng liên tỉnh với ràng buộc tuyến đường và tải trọng

**Repository:** `ride-hailing-freight-route-optimization`

**Hệ thống:** Một platform kết nối chủ hàng với xe tải vận chuyển liên tỉnh, mỗi xe có tải trọng và tuyến đường cố định, chủ hàng đăng đơn hàng cần vận chuyển theo tuyến.

**Vai trò của flow:** Flow matching phải gán đơn hàng cho xe tải phù hợp (đủ tải trọng còn lại, đúng tuyến), xử lý đúng khi nhiều đơn hàng nhỏ cùng cạnh tranh phần tải trọng còn lại của 1 xe.

**Yêu cầu cụ thể:**
- Tải trọng còn lại của mỗi xe phải được trừ atomic ngay khi 1 đơn hàng được xác nhận gán vào xe đó (`UPDATE remaining_capacity = remaining_capacity - weight WHERE remaining_capacity >= weight`), không đọc-tính-ghi ở application để tránh 2 đơn hàng cùng được gán vào phần tải trọng cuối cùng còn thiếu.
- Mô tả cụ thể: xe còn 500kg tải trọng, 2 đơn hàng 300kg và 400kg được đề xuất gán gần như đồng thời (tổng vượt quá 500kg) — chỉ đúng 1 đơn được gán thành công theo thứ tự request xử lý atomic tới trước, đơn còn lại phải được matching engine tự động tìm xe khác thay thế, không bị "treo" chờ vô thời hạn.
- Khi chủ xe cập nhật lại tải trọng khả dụng (ví dụ phát hiện xe cần chở thêm hàng dự trữ, giảm tải trọng cho thuê) đúng lúc có đơn hàng đang được gán vào xe đó, quy định rõ đơn đã gán trước khi cập nhật vẫn giữ nguyên, chỉ ảnh hưởng tới các đơn mới sau đó — tránh việc "giật" đơn đã gán ra khỏi xe do cập nhật tải trọng giữa lúc xử lý.
- Nếu 1 đơn hàng lớn cần chia cho 2 xe khác nhau (do 1 xe không đủ tải trọng), việc gán từng phần vào từng xe phải là các transaction atomic riêng biệt nhưng có ràng buộc: nếu 1 phần gán thất bại (không tìm được xe phù hợp) thì phần đã gán ở xe khác phải được rollback hoặc có cơ chế bù trừ (không để đơn hàng bị chia dở dang không rõ trạng thái).
- Có validate route matching atomic cùng lúc với validate tải trọng: 1 đơn hàng chỉ được gán vào xe nếu cả 2 điều kiện đều thỏa tại đúng thời điểm gán (không kiểm tra tuyến trước rồi kiểm tra tải trọng sau ở 2 bước tách biệt, dễ có khoảng hở khi dữ liệu xe thay đổi giữa 2 bước).

---

## Điều phối xe cấp cứu ưu tiên chuyến khẩn cấp giữa nhiều yêu cầu cùng lúc

**Repository:** `ride-hailing-ambulance-priority-dispatch`

**Hệ thống:** Một hệ thống điều phối xe cấp cứu cho bệnh viện/dịch vụ y tế, nhận yêu cầu gọi xe cấp cứu từ nhiều nguồn (gọi tổng đài, app, bệnh viện chuyển viện) và phải ưu tiên đúng theo mức độ khẩn cấp.

**Vai trò của flow:** Flow matching phải chọn đúng xe cấp cứu gần nhất còn trống cho ca khẩn cấp nhất khi có nhiều yêu cầu đến gần như đồng thời, đảm bảo không xe nào bị gán trùng 2 ca và ca khẩn cấp hơn không bị xe đi phục vụ ca ít khẩn cấp hơn trước.

**Yêu cầu cụ thể:**
- Khi có nhiều yêu cầu đến trong cùng 1 khung thời gian ngắn, việc chọn xe phải ưu tiên theo mức độ khẩn cấp trước, sau đó theo khoảng cách — mô tả rõ transaction gán xe phải xử lý theo đúng thứ tự ưu tiên này (không đơn giản theo thứ tự request đến trước), có thể cần 1 hàng đợi ưu tiên (priority queue) atomic thay vì xử lý FIFO đơn giản.
- Mô tả cụ thể: xe cấp cứu A đang được xem xét gán cho ca "gãy tay" (ưu tiên thấp) đúng lúc có ca "đột quỵ" (ưu tiên cao) mới xuất hiện gần xe A hơn tất cả xe khác — nếu xe A CHƯA được xác nhận gán (còn trong quá trình chọn), hệ thống phải ưu tiên gán xe A cho ca đột quỵ, đẩy ca gãy tay sang xe khác; nhưng nếu xe A ĐÃ được xác nhận gán (transaction đã hoàn tất) cho ca gãy tay, không được "giật" lại xe đang trên đường xử lý ca đó.
- Đảm bảo việc gán xe là atomic tuyệt đối (không có khoảng hở giữa "chọn xe" và "xác nhận gán") để tránh trường hợp 2 case khẩn cấp cùng chọn trúng cùng 1 xe do đọc trạng thái "đang trống" gần như đồng thời trước khi 1 trong 2 kịp cập nhật trạng thái.
- Khi xe cấp cứu báo "đang bận" (do gặp tình huống ngoài dự kiến trên đường) đúng lúc đang được matching engine xem xét gán cho ca mới, phải có cơ chế cập nhật trạng thái xe real-time và transaction gán phải re-check trạng thái ngay trước khi xác nhận, không dựa vào trạng thái đã đọc từ vài giây trước.
- Ghi log đầy đủ toàn bộ quyết định điều phối (ca nào, xe nào, tại sao chọn, có bị đẩy ưu tiên không) để phục vụ kiểm tra/audit sau này, đặc biệt quan trọng với hệ thống y tế cần minh bạch quyết định.

---

## Ghép chuyến đi chung (carpooling) giữa nhiều khách có điểm đón/trả gần nhau

**Repository:** `ride-hailing-carpooling-matching`

**Hệ thống:** Một app đi chung xe (carpool) ghép nhiều khách có lộ trình gần nhau vào cùng 1 chuyến xe để chia sẻ chi phí, tài xế nhận tối đa 1 số khách nhất định theo số ghế trống.

**Vai trò của flow:** Flow matching phải ghép đúng khách vào chuyến xe có đủ ghế trống và lộ trình phù hợp, xử lý đúng khi nhiều khách cùng được đề xuất ghép vào cùng 1 chuyến gần như đồng thời và số ghế có hạn.

**Yêu cầu cụ thể:**
- Số ghế trống của mỗi chuyến phải được trừ atomic ngay khi có khách xác nhận ghép chuyến (`UPDATE seats_available = seats_available - 1 WHERE seats_available > 0`), không đọc-tính-ghi, để tránh 2 khách cùng ghép vào ghế trống cuối cùng.
- Mô tả cụ thể: chuyến xe còn 1 ghế trống, 2 khách khác nhau nhận được đề xuất ghép chuyến gần như đồng thời (do matching engine tính toán độc lập cho từng khách trong cùng khung thời gian) và cả 2 đều bấm xác nhận — chỉ đúng 1 khách ghép thành công theo transaction atomic tới trước, khách còn lại phải tự động được matching engine đề xuất chuyến khác ngay, không hiển thị lỗi cụt mà không có gợi ý tiếp theo.
- Khi 1 khách đã ghép vào chuyến rồi hủy ngay trước giờ khởi hành, ghế trống phải được trả lại atomic và hệ thống phải tái chạy matching cho các khách khác đang chờ ghép gần tuyến đó ngay lập tức (không chờ tới lượt quét job định kỳ tiếp theo, vì thời gian khởi hành gần nên cần phản ứng nhanh).
- Việc tính toán lộ trình tối ưu khi ghép thêm 1 khách mới (điểm đón/trả có thể làm thay đổi lộ trình của khách đã ghép trước) phải được xác nhận atomic cùng lúc với việc trừ ghế trống — không trừ ghế trước rồi tính lộ trình sau (nếu lộ trình mới không khả thi về thời gian, phải rollback cả việc trừ ghế).
- Nếu tài xế hủy chuyến (ví dụ đổi ý không chạy nữa) đúng lúc có khách đang trong quá trình xác nhận ghép chuyến đó, request xác nhận ghép phải bị chặn ngay nếu chuyến đã chuyển sang trạng thái "đã hủy" tại thời điểm transaction ghép chạy, và khách phải được thông báo rõ để tìm chuyến khác, không rơi vào trạng thái "đã ghép" vào 1 chuyến không còn tồn tại.
