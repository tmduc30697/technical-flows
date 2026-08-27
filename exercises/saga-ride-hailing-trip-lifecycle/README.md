# Điều phối vòng đời một chuyến đi ride-hailing qua nhiều bước

**Hệ thống:** App gọi xe điều phối một chuyến đi qua chuỗi bước: tìm tài xế phù hợp → tài xế nhận cuốc → tài xế đón khách → hoàn thành chuyến đi → thanh toán, mỗi bước xảy ra ở thiết bị khách/tài xế khác nhau và có thể mất kết nối bất kỳ lúc nào.

**Vai trò của flow:** Saga theo dõi và điều phối trạng thái chuyến đi xuyên các service (matching, trip, payment), xử lý hủy/mất kết nối giữa chừng ở từng bước và đưa chuyến đi về trạng thái nhất quán thay vì kẹt lửng lơ.

**Yêu cầu cụ thể:**
- Nếu tài xế đã "nhận cuốc" nhưng sau đó tự hủy trước khi đón khách, saga phải compensate bằng cách trả chuyến đi về trạng thái "tìm tài xế" và loại trừ tài xế vừa hủy khỏi vòng matching tiếp theo cho cùng chuyến, tránh matching lặp lại ngay với tài xế vừa từ chối.
- Khi app khách mất kết nối đúng lúc tài xế đang trên đường đón (giữa bước "nhận cuốc" và "đón khách"), saga không được tự hủy chuyến ngay lập tức — cần cửa sổ chờ hợp lý để khách reconnect, vì hủy sai có thể khiến tài xế đã di chuyển gần tới nơi mất công vô ích và ảnh hưởng đánh giá.
- Race condition khi khách hủy chuyến đúng lúc tài xế bấm "đã đón khách" gửi lên gần như đồng thời — hai sự kiện đến từ hai thiết bị khác nhau qua network độc lập, saga phải có luật ưu tiên rõ ràng (ví dụ trạng thái nào server nhận trước theo thời điểm xử lý được coi là hợp lệ) và bên thua phải nhận thông báo đúng trạng thái thực tế, không để hai app hiển thị hai trạng thái mâu thuẫn.
- Bước thanh toán ở cuối saga có thể thất bại (thẻ bị từ chối) sau khi chuyến đã hoàn thành thực tế — saga không được compensate bằng cách "hủy chuyến" vì chuyến đã xảy ra không thể hủy ngược, mà phải tách thành trạng thái "hoàn thành, chờ xử lý thanh toán" với luồng retry/thu nợ riêng, đồng thời vẫn ghi nhận thu nhập tạm cho tài xế.
- Saga phải resume đúng bước khi service điều phối trip bị restart giữa lúc chuyến đang chạy (ví dụ deploy giữa lúc có hàng nghìn chuyến đang diễn ra) — trạng thái từng chuyến phải persist đủ để không gửi lại thông báo "đã tìm thấy tài xế" cho chuyến đã qua bước đó từ trước, gây nhiễu loạn app khách/tài xế.
