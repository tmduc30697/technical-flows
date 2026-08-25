# Flash sale/high-concurrency checkout flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống bán hàng theo sự kiện (flash sale e-commerce, bán giày/đồ hiếm giới hạn, bán vé sự kiện, bán suất ưu đãi ngân hàng) để luyện xử lý lượng request đồng thời cực lớn đổ vào một số lượng tài nguyên rất hạn chế tại đúng 1 thời điểm mở bán.

---

## Flash sale số lượng giới hạn trên sàn e-commerce (100 sản phẩm, hàng chục nghìn người chờ)

**Repository:** `flash-sale-ecommerce-limited-stock`

**Hệ thống:** Một sàn e-commerce tổ chức flash sale: đúng 12:00:00 mở bán 100 sản phẩm giá sốc, dự kiến hàng chục nghìn người cùng bấm "Mua ngay" trong vài giây đầu.

**Vai trò của flow:** Flow checkout phải đảm bảo đúng 100 người mua được (không hơn không kém), xử lý đúng khi hàng chục nghìn request cùng cạnh tranh vào đúng thời điểm mở bán, không để hệ thống sập hoặc bán vượt tồn kho (overselling).

**Yêu cầu cụ thể:**
- Không cho phép dùng `SELECT tồn_kho rồi kiểm tra > 0 rồi UPDATE trừ` ở tầng application (race condition kinh điển: nhiều request đọc thấy tồn kho > 0 cùng lúc rồi đều trừ) — yêu cầu dùng update nguyên tử kiểu `UPDATE stock SET qty = qty - 1 WHERE qty > 0` và kiểm tra số dòng bị ảnh hưởng (affected rows) để biết có mua được hay không.
- Thiết kế hàng đợi (queue) hoặc rate-limiter ở tầng trước DB để giảm áp lực: chỉ cho một số lượng request giới hạn/giây chạm vào transaction trừ tồn kho, các request còn lại nhận phản hồi "đang xử lý, vui lòng chờ" thay vì để tất cả cùng đấm vào DB gây timeout hàng loạt.
- Mô tả cụ thể: 2 request đến gần như đồng thời tại thời điểm tồn kho còn đúng 1 sản phẩm — hệ thống phải đảm bảo chỉ 1 request được xác nhận mua, request kia nhận lỗi "hết hàng" trong thời gian ngắn (không phải timeout), viết test giả lập 1000 request đồng thời cho 100 sản phẩm và assert đúng 100 request thành công.
- Idempotency: nếu client retry do timeout mạng (không rõ request trước có xử lý thành công hay không), request retry với cùng idempotency key không được trừ tồn kho lần 2 — phải trả lại đúng kết quả của lần xử lý đầu.
- Có cơ chế "circuit breaker"/giảm tải khi tồn kho đã về 0: các request đến sau phải được chặn ngay từ tầng cache/API gateway, không để lọt xuống DB gây tải vô ích.

---

## Bán giày phiên bản giới hạn (sneaker drop) chống bot mua sạch hàng

**Repository:** `flash-sale-sneaker-drop-anti-bot`

**Hệ thống:** Một nền tảng bán giày sneaker limited edition, mở bán 500 cặp vào giờ cố định, biết trước sẽ có bot cố gắng mua hết để bán lại (scalping).

**Vai trò của flow:** Flow checkout phải phân biệt được (ở mức hợp lý) request người dùng thật với bot, đồng thời vẫn đảm bảo tính đúng đắn của việc trừ tồn kho khi hàng nghìn request hợp lệ cạnh tranh cùng lúc.

**Yêu cầu cụ thể:**
- Giới hạn mỗi tài khoản/mỗi địa chỉ giao hàng chỉ được mua tối đa 1 cặp giày trong sự kiện — yêu cầu ràng buộc unique ở tầng DB (không chỉ check ở application) trên (event_id, user_id) để loại trừ race khi cùng 1 user gửi nhiều request mua đồng thời từ nhiều thiết bị/tab.
- Thiết kế cơ chế captcha/challenge trước khi vào hàng đợi mua, nhưng vẫn phải đảm bảo transaction trừ tồn kho phía sau atomic bất kể request đến từ nguồn nào — mô tả rõ 2 lớp bảo vệ này độc lập với nhau (chống bot không thay thế cho việc chống race condition).
- Mô tả cụ thể: 1 user vượt qua challenge và gửi đúng 1 request mua, nhưng do mạng lag client tự động retry gửi thêm 2 request giống nhau trong vòng 1 giây — hệ thống phải nhận diện đây là request trùng (qua idempotency key hoặc unique constraint) và chỉ xử lý 1 lần, không trừ tồn kho 3 lần hay charge thẻ 3 lần.
- Khi tồn kho giảm về 0, mọi request đang chờ trong hàng đợi phải được trả kết quả "hết hàng" trong một khoảng thời gian giới hạn rõ ràng (không để user chờ vô thời hạn không biết kết quả).
- Có cơ chế theo dõi (rate limit theo IP/device fingerprint) để phát hiện 1 nguồn gửi số lượng request bất thường trong sự kiện, ghi log để phân tích sau, không chặn cứng ngay lập tức gây ảnh hưởng người dùng thật dùng chung IP (ví dụ mạng công ty/wifi công cộng).

---

## Mở bán suất vay ưu đãi lãi suất thấp giới hạn số lượng của ngân hàng số

**Repository:** `flash-sale-digital-bank-loan-offer`

**Hệ thống:** Một app ngân hàng số mở chương trình vay tiêu dùng lãi suất ưu đãi, giới hạn 1000 suất, khách hàng phải bấm đăng ký ngay khi mở, ai đăng ký trước và đủ điều kiện được duyệt trước.

**Vai trò của flow:** Flow đăng ký phải xử lý đúng thứ tự "đến trước được trước" khi có lượng lớn khách hàng bấm đăng ký cùng lúc, đồng thời phải chạy kiểm tra điều kiện tín dụng (gọi hệ thống bên ngoài, có độ trễ) mà không làm sai thứ tự ưu tiên.

**Yêu cầu cụ thể:**
- Vì kiểm tra điều kiện tín dụng có độ trễ (vài giây, gọi credit bureau), không thể trừ suất ngay khi bấm đăng ký — yêu cầu thiết kế 2 bước: (1) giữ "vị trí hàng đợi" atomic ngay khi bấm đăng ký (theo timestamp/counter tăng dần), (2) xác nhận suất chỉ sau khi qua kiểm tra tín dụng, và nếu không đạt điều kiện phải nhả suất cho người kế tiếp trong hàng đợi.
- Mô tả cụ thể: khách hàng #999 và #1000 giữ được vị trí hợp lệ, nhưng khách #999 bị từ chối tín dụng sau 5 giây — suất của #999 phải được chuyển đúng cho khách #1001 (người xếp hàng kế tiếp), không phải bỏ trống suất đó, và toàn bộ việc "nhả suất - cấp suất kế tiếp" phải atomic để không có 2 khách cùng được cấp 1 suất.
- Đảm bảo bộ đếm vị trí hàng đợi (counter) tăng đúng tuần tự dưới tải cao (dùng cơ chế atomic increment ở DB hoặc cache, không dùng đọc-rồi-tăng ở application vì sẽ mất increment khi nhiều request song song).
- Nếu request kiểm tra tín dụng bị timeout (hệ thống bên ngoài chậm), quy định retry tối đa bao nhiêu lần và trong thời gian bao lâu trước khi coi là thất bại và nhả suất cho người kế tiếp — không để 1 khách timeout chặn vô thời hạn cả hàng đợi phía sau.
- Có dashboard hiển thị real-time số suất còn lại và vị trí hàng đợi hiện tại cho khách đang chờ, đồng bộ đúng với trạng thái thật trong DB (không hiển thị số liệu cache trễ gây hiểu nhầm đã hết suất khi thực ra còn).
