# Bán giày phiên bản giới hạn (sneaker drop) chống bot mua sạch hàng

**Hệ thống:** Một nền tảng bán giày sneaker limited edition, mở bán 500 cặp vào giờ cố định, biết trước sẽ có bot cố gắng mua hết để bán lại (scalping).

**Vai trò của flow:** Flow checkout phải phân biệt được (ở mức hợp lý) request người dùng thật với bot, đồng thời vẫn đảm bảo tính đúng đắn của việc trừ tồn kho khi hàng nghìn request hợp lệ cạnh tranh cùng lúc.

**Yêu cầu cụ thể:**
- Giới hạn mỗi tài khoản/mỗi địa chỉ giao hàng chỉ được mua tối đa 1 cặp giày trong sự kiện — yêu cầu ràng buộc unique ở tầng DB (không chỉ check ở application) trên (event_id, user_id) để loại trừ race khi cùng 1 user gửi nhiều request mua đồng thời từ nhiều thiết bị/tab.
- Thiết kế cơ chế captcha/challenge trước khi vào hàng đợi mua, nhưng vẫn phải đảm bảo transaction trừ tồn kho phía sau atomic bất kể request đến từ nguồn nào — mô tả rõ 2 lớp bảo vệ này độc lập với nhau (chống bot không thay thế cho việc chống race condition).
- Mô tả cụ thể: 1 user vượt qua challenge và gửi đúng 1 request mua, nhưng do mạng lag client tự động retry gửi thêm 2 request giống nhau trong vòng 1 giây — hệ thống phải nhận diện đây là request trùng (qua idempotency key hoặc unique constraint) và chỉ xử lý 1 lần, không trừ tồn kho 3 lần hay charge thẻ 3 lần.
- Khi tồn kho giảm về 0, mọi request đang chờ trong hàng đợi phải được trả kết quả "hết hàng" trong một khoảng thời gian giới hạn rõ ràng (không để user chờ vô thời hạn không biết kết quả).
- Có cơ chế theo dõi (rate limit theo IP/device fingerprint) để phát hiện 1 nguồn gửi số lượng request bất thường trong sự kiện, ghi log để phân tích sau, không chặn cứng ngay lập tức gây ảnh hưởng người dùng thật dùng chung IP (ví dụ mạng công ty/wifi công cộng).
