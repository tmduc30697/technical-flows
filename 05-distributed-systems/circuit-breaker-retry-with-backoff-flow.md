# Circuit breaker & retry with backoff flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần bảo vệ hệ thống khi gọi dịch vụ bên ngoài không ổn định — payment gateway, API vận chuyển, LLM/AI provider, API bản đồ cho ride-hailing, và core banking legacy — nhằm luyện cách cấu hình ngưỡng mở circuit breaker, retry an toàn với backoff/jitter, và fallback hợp lý trong web app thực tế.

---

## Checkout gọi payment gateway bên thứ ba

**Repository:** `circuit-breaker-checkout-payment-gateway`

**Hệ thống:** Trang e-commerce gọi API cổng thanh toán bên ngoài (kiểu Stripe) để xử lý thanh toán lúc checkout.

**Vai trò của flow:** Circuit breaker + retry with backoff bảo vệ hệ thống khi payment gateway chậm/lỗi, tránh làm nghẽn toàn bộ luồng checkout và tránh gọi lại dồn dập làm gateway càng quá tải.

**Yêu cầu cụ thể:**
- Định nghĩa rõ ngưỡng mở circuit breaker (ví dụ tỉ lệ lỗi >50% trong 10 giây gần nhất, hoặc latency p95 vượt X ms) và hành vi khi breaker mở (fail-fast, không gọi gateway nữa trong một khoảng thời gian).
- Retry chỉ áp dụng cho lỗi có thể retry an toàn (timeout, 5xx) — tuyệt đối không retry tự động cho response không rõ kết quả liên quan tới việc charge tiền (tránh double-charge), phải có bước kiểm tra idempotency key trước khi retry.
- Backoff phải là exponential kèm jitter (ngẫu nhiên hóa) để tránh nhiều request cùng retry đồng loạt tạo ra "retry storm" làm gateway sập thêm.
- Circuit breaker phải có trạng thái half-open để thử lại một lượng nhỏ request sau khi hết thời gian mở, và chỉ đóng lại hoàn toàn khi các request thử nghiệm đó thành công.
- Đo lường: tỉ lệ request bị fail-fast do breaker mở, số lần breaker chuyển trạng thái trong ngày, và alert khi breaker mở quá lâu (báo hiệu sự cố kéo dài ở phía gateway).

---

## SaaS gọi LLM/AI provider bên thứ ba cho tính năng chat AI

**Repository:** `circuit-breaker-llm-provider-chat-ai`

**Hệ thống:** Một SaaS có tính năng chat AI gọi API của nhà cung cấp mô hình ngôn ngữ bên ngoài để trả lời user.

**Vai trò của flow:** Circuit breaker + retry giúp xử lý khi provider AI bị rate limit, timeout, hoặc quá tải, tránh trải nghiệm user bị treo vô thời hạn hoặc gọi lại làm tốn quota/tiền không cần thiết.

**Yêu cầu cụ thể:**
- Retry chỉ thực hiện với lỗi timeout/5xx và tối đa số lần retry giới hạn (ví dụ 2 lần) với backoff, vì mỗi lần gọi provider AI có chi phí (tính theo token) — retry vô tội vạ gây tốn tiền.
- Khi provider trả lỗi 429 (rate limit), phải đọc header retry-after (nếu có) để backoff đúng thời gian được yêu cầu, không tự đoán.
- Circuit breaker mở khi tỉ lệ lỗi/timeout của provider vượt ngưỡng, và hệ thống phải có fallback rõ ràng khi breaker mở (ví dụ chuyển sang provider dự phòng thứ hai, hoặc trả thông báo lỗi thân thiện cho user thay vì treo loading vô hạn).
- Phải phân biệt và không retry với lỗi do chính request của user sai (ví dụ input vượt quá giới hạn token, prompt bị filter) — retry những lỗi này chỉ tốn thời gian mà không giải quyết được gì.
- Đo lường: latency thêm vào do retry (retry overhead), tỉ lệ request phải fallback sang provider dự phòng, và chi phí phát sinh do retry để đưa vào theo dõi ngân sách AI.

---

## Ngân hàng số gọi hệ thống core banking nội bộ (legacy)

**Repository:** `circuit-breaker-digital-bank-legacy-core`

**Hệ thống:** App ngân hàng số hiện đại gọi vào hệ thống core banking cũ (legacy, hiệu năng hạn chế) để lấy số dư, thực hiện giao dịch.

**Vai trò của flow:** Circuit breaker bảo vệ core banking (rất khó scale, không chịu được tải đột biến) khỏi bị app hiện đại gọi dồn dập lúc cao điểm hoặc lúc core banking đang chậm.

**Yêu cầu cụ thể:**
- Circuit breaker phải cấu hình chặt hơn bình thường (ngưỡng mở thấp hơn) để bảo vệ core banking vốn dễ bị sập khi quá tải, ưu tiên bảo vệ hệ thống lõi hơn là giữ app luôn phản hồi.
- Retry cho giao dịch ghi (chuyển tiền, thanh toán) phải cực kỳ cẩn trọng: chỉ retry khi chắc chắn core banking chưa xử lý request trước đó (dựa vào transaction reference được core banking hỗ trợ tra cứu trạng thái), tuyệt đối không retry mù.
- Với giao dịch đọc (xem số dư), có thể fallback về số dư cache gần nhất khi breaker mở, nhưng phải hiển thị rõ cho user đây là "số dư tại thời điểm gần nhất" không phải real-time.
- Khi breaker mở kéo dài (core banking down lâu), phải có quy trình escalate tới team vận hành ngay (alert khẩn) vì đây ảnh hưởng trực tiếp tới khả năng giao dịch của khách hàng.
- Ghi log đầy đủ mọi lần retry/circuit breaker chuyển trạng thái liên quan tới giao dịch tiền, phục vụ audit và đối soát khi có khiếu nại từ khách hàng.
