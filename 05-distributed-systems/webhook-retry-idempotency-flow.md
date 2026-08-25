# Webhook retry & Idempotency flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần gửi webhook đáng tin cậy — payment gateway gửi merchant, e-commerce gửi đối tác vận chuyển, SaaS gửi endpoint khách hàng, chat platform gửi bot backend, và marketplace gửi hệ thống kế toán — nhằm luyện cách retry với backoff, ký/verify signature, và đảm bảo idempotency ở phía nhận trong web app thực tế.

---

## Payment gateway gửi webhook cho merchant (kiểu Stripe)

**Repository:** `webhook-retry-payment-gateway-stripe-like`

**Hệ thống:** Một cổng thanh toán (đóng vai trò gửi webhook) cần thông báo cho hệ thống merchant khi trạng thái thanh toán thay đổi (thành công, thất bại, hoàn tiền).

**Vai trò của flow:** Đảm bảo merchant nhận được webhook dù endpoint của họ tạm thời down, và merchant xử lý đúng dù nhận trùng webhook (do retry) mà không bị double-apply.

**Yêu cầu cụ thể:**
- Cổng thanh toán phải retry gửi webhook với exponential backoff khi merchant endpoint trả lỗi/timeout, có giới hạn tổng thời gian retry rõ ràng (ví dụ retry trong vòng 24 giờ rồi dừng và đánh dấu failed).
- Mỗi webhook phải có một event ID duy nhất và không đổi qua các lần retry, để merchant có thể dùng làm khóa idempotency (lưu lại event ID đã xử lý, bỏ qua nếu nhận trùng).
- Webhook payload phải được ký (HMAC signature) để merchant xác thực đúng là từ cổng thanh toán gửi, chống giả mạo webhook.
- Merchant phải trả response (2xx) thật nhanh sau khi đã lưu nhận webhook (queue xử lý async), không xử lý nghiệp vụ nặng ngay trong request webhook — tránh timeout khiến cổng thanh toán coi là thất bại và retry không cần thiết.
- Có trang/API cho merchant xem lại lịch sử webhook đã gửi, trạng thái (đã nhận/thất bại/đang retry), và cho phép merchant yêu cầu gửi lại thủ công (replay) một webhook cụ thể.

---

## Nền tảng e-commerce gửi webhook trạng thái đơn hàng cho đối tác vận chuyển

**Repository:** `webhook-retry-ecommerce-shipping-partner`

**Hệ thống:** Sàn e-commerce gửi webhook cho hệ thống của đối tác vận chuyển mỗi khi có đơn hàng mới cần lấy/giao.

**Vai trò của flow:** Đảm bảo đối tác vận chuyển không bỏ lỡ đơn hàng dù hệ thống của họ gặp sự cố tạm thời, và không tạo đơn ship trùng khi webhook bị gửi lại.

**Yêu cầu cụ thể:**
- Retry webhook phải dừng lại và chuyển sang cơ chế cảnh báo (email/dashboard) nếu đối tác liên tục fail sau X lần retry trong khoảng thời gian dài, để tránh gửi vô tận vào một endpoint đã chết hẳn.
- Đối tác vận chuyển phải implement idempotency dựa trên order ID + event type để xử lý đúng khi nhận 2 webhook giống nhau (do sàn gửi lại vì chưa nhận được ack lần trước dù thực tế đã xử lý thành công).
- Webhook phải mang theo thứ tự sự kiện (sequence number hoặc timestamp) để đối tác phát hiện nhận sai thứ tự (ví dụ nhận "đã hủy" trước "đã tạo" do network delay) và xử lý hợp lý (không apply ngược thứ tự).
- Sàn e-commerce phải có cơ chế theo dõi tỉ lệ thành công gửi webhook theo từng đối tác, cảnh báo khi tỉ lệ lỗi tăng bất thường (dấu hiệu đối tác đang gặp sự cố hệ thống).
- Có endpoint cho phép đối tác chủ động pull lại các event bị lỡ (dùng để đối soát/backfill) trong trường hợp nghi ngờ có event bị mất mà không qua webhook push.

---

## SaaS platform gửi webhook tới hệ thống của khách hàng (customer-configured endpoint)

**Repository:** `webhook-retry-saas-customer-endpoint`

**Hệ thống:** Một SaaS B2B cho phép khách hàng tự khai báo endpoint webhook để nhận thông báo khi có sự kiện trong tài khoản của họ (ví dụ có user mới, có report hoàn tất).

**Vai trò của flow:** Đảm bảo độ tin cậy gửi webhook tới hàng nghìn endpoint khách hàng khác nhau với độ ổn định rất khác nhau, không để một khách hàng cấu hình endpoint chậm/lỗi ảnh hưởng tới việc gửi webhook cho khách khác.

**Yêu cầu cụ thể:**
- Việc gửi webhook cho từng khách hàng phải cách ly (isolated) — endpoint của khách A bị timeout liên tục không được làm chậm/nghẽn hàng đợi gửi webhook cho khách B.
- Retry phải theo backoff tăng dần và có trần tối đa số lần thử/tổng thời gian, sau đó tự động tạm dừng (auto-disable) webhook của khách đó và thông báo cho họ qua email/dashboard để tự kiểm tra endpoint.
- Cung cấp cho khách hàng một secret riêng để verify signature webhook, và hướng dẫn rõ cách chống replay attack (kiểm tra timestamp trong payload không quá cũ).
- Cho khách hàng self-service xem log lịch sử gửi (trạng thái, response code, thời gian) và có nút "test webhook"/"gửi lại" mà không cần liên hệ support.
- Đo lường SLA gửi webhook nội bộ (ví dụ 99% webhook được gửi thành công trong vòng 5 giây kể từ khi event xảy ra) và alert khi SLA không đạt.

---

## Chat platform gửi webhook cho bot backend của bên thứ ba

**Repository:** `webhook-retry-chat-platform-bot`

**Hệ thống:** Nền tảng chat (kiểu Slack) cho phép bên thứ ba đăng ký bot, mỗi khi có message/mention platform gửi webhook cho bot backend xử lý và trả lời.

**Vai trò của flow:** Đảm bảo bot backend nhận đủ event để phản hồi đúng ngữ cảnh, xử lý được trường hợp webhook trùng/trễ ảnh hưởng tới trải nghiệm hội thoại tự động.

**Yêu cầu cụ thể:**
- Webhook phải được gửi và xử lý với độ trễ rất thấp (dưới 1-2 giây) vì ảnh hưởng trực tiếp tới trải nghiệm hội thoại real-time — nêu rõ chiến lược timeout/retry ngắn phù hợp use case chat, khác với webhook nghiệp vụ chậm khác.
- Bot backend nhận trùng webhook (do platform gửi lại vì chưa nhận ack đúng hạn) không được trả lời lặp lại 2 lần cùng một message — dùng message ID làm khóa idempotency để chặn xử lý trùng.
- Nếu bot backend down một khoảng thời gian ngắn, platform phải retry trong một window ngắn hợp lý (ví dụ vài chục giây) — quá thời gian đó thì bỏ, không nên cố gửi tin nhắn cũ trả lời trễ nhiều phút gây khó hiểu cho người dùng.
- Xử lý được trường hợp 2 webhook liên quan tới cùng một thread đến sai thứ tự (message sau đến trước message trước) — bot phải dựa vào timestamp/thread order thực sự, không dựa vào thứ tự nhận webhook.
- Có dashboard cho nhà phát triển bot theo dõi tỉ lệ webhook delivery thành công/trễ, giúp họ debug khi bot "không phản hồi" do vấn đề ở tầng webhook chứ không phải logic bot.

---

## Marketplace gửi webhook cho hệ thống kế toán/accounting của seller

**Repository:** `webhook-retry-marketplace-seller-accounting`

**Hệ thống:** Marketplace cần gửi thông báo mỗi khi có giao dịch phát sinh (bán hàng, hoàn tiền, phí) tới hệ thống kế toán mà seller tự kết nối.

**Vai trò của flow:** Đảm bảo mọi sự kiện tài chính đều đến được hệ thống kế toán của seller đúng một lần logic, vì đây là dữ liệu ảnh hưởng trực tiếp tới sổ sách tài chính — sai sót ở đây nghiêm trọng hơn webhook thông thường.

**Yêu cầu cụ thể:**
- Mỗi webhook tài chính phải có transaction ID toàn cục duy nhất và không thay đổi qua các lần gửi lại, để hệ thống kế toán của seller ghi nhận đúng một lần bất kể nhận webhook bao nhiêu lần.
- Cung cấp API cho phép seller chủ động query lại toàn bộ event tài chính trong một khoảng thời gian (reconciliation API), để họ tự đối soát nếu nghi ngờ thiếu event từ webhook.
- Retry gửi webhook phải kéo dài hơn webhook thông thường (ví dụ vài ngày) vì đây là dữ liệu quan trọng không thể chấp nhận mất, kèm alert nội bộ khi một seller liên tục fail nhận webhook tài chính.
- Đảm bảo thứ tự các webhook liên quan tới cùng một giao dịch (ví dụ "tạo" rồi "hoàn tiền") được gửi và có thể được xử lý đúng thứ tự ở phía seller (đánh số thứ tự hoặc timestamp đáng tin cậy).
- Log đầy đủ và không thể sửa (immutable) lịch sử gửi webhook tài chính cho mục đích audit, có thể truy vết lại bất kỳ lúc nào khi có tranh chấp giữa marketplace và seller.
