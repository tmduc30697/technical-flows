# Transactional Outbox pattern flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần phát event đáng tin cậy cùng transaction ghi DB — order service e-commerce, notification service, payment service, đồng bộ CRM, và inventory service — nhằm luyện cách tránh dual-write problem, đảm bảo publish at-least-once và idempotency ở consumer trong web app thực tế.

---

## Order service phát event sau khi tạo đơn hàng (e-commerce)

**Repository:** `outbox-ecommerce-order-event`

**Hệ thống:** Order service ghi đơn hàng vào DB và cần bắn event "OrderCreated" cho các service khác (inventory, notification, analytics) qua message broker.

**Vai trò của flow:** Transactional Outbox đảm bảo việc ghi đơn hàng vào DB và việc "chuẩn bị gửi event" xảy ra trong cùng một transaction DB, tránh dual-write problem (ghi DB thành công nhưng event bị mất, hoặc ngược lại).

**Yêu cầu cụ thể:**
- Việc insert record đơn hàng và insert record vào bảng outbox phải nằm trong cùng một database transaction (commit hoặc rollback cùng lúc).
- Một relay process (poller hoặc CDC) đọc bảng outbox và publish lên message broker, sau đó đánh dấu record outbox là "đã publish" — không xóa ngay để phục vụ audit/debug.
- Đảm bảo publish là "at-least-once": nếu relay crash sau khi publish nhưng trước khi đánh dấu, khi restart phải publish lại — consumer phía sau phải tự xử lý idempotency (dựa vào event ID).
- Outbox table phải có cơ chế dọn dẹp record cũ đã publish thành công (archive/xóa sau X ngày) để không phình vô hạn.
- Đo lường: độ trễ từ lúc transaction DB commit tới lúc event thực sự lên broker (outbox lag), và alert khi lag vượt ngưỡng (ví dụ >30s).

---

## Notification service đảm bảo gửi email/SMS đúng 1 lần logic sau hành động nghiệp vụ

**Repository:** `outbox-notification-exactly-once`

**Hệ thống:** Một service xử lý các hành động (đăng ký, đổi mật khẩu, thanh toán) cần trigger gửi email/SMS thông báo tương ứng.

**Vai trò của flow:** Outbox pattern lưu "ý định gửi thông báo" cùng transaction với hành động nghiệp vụ, tách rời việc gọi email/SMS provider (có thể chậm/lỗi) khỏi transaction chính.

**Yêu cầu cụ thể:**
- Transaction ghi hành động nghiệp vụ (ví dụ đổi mật khẩu) và ghi outbox record "cần gửi email xác nhận" phải atomic — không có trường hợp đổi mật khẩu thành công nhưng mất luôn ý định gửi email.
- Relay gửi email không được block transaction chính; nó chạy nền và đọc outbox theo batch định kỳ (ví dụ mỗi 2 giây một lần).
- Nếu provider email/SMS trả lỗi tạm thời, outbox record phải được retry với backoff, và có giới hạn số lần retry trước khi đánh dấu "failed" và alert.
- Không được gửi trùng thông báo cho cùng một outbox record nếu relay bị restart giữa lúc gửi — dùng cờ trạng thái (pending/sending/sent/failed) với điều kiện cập nhật an toàn (optimistic lock) để tránh 2 relay instance cùng gửi.
- Có dashboard hiển thị số lượng thông báo đang pending, tỉ lệ thất bại theo provider, để phát hiện provider đang gặp sự cố.

---

## Payment service ghi ledger nội bộ và publish event cho hệ thống kế toán

**Repository:** `outbox-payment-ledger-accounting-event`

**Hệ thống:** Payment service xử lý giao dịch, cần vừa ghi vào ledger nội bộ vừa gửi event cho hệ thống kế toán/báo cáo bên ngoài.

**Vai trò của flow:** Outbox pattern bảo đảm event kế toán không bao giờ "biến mất" dù broker tạm thời không khả dụng lúc giao dịch xảy ra, vì event luôn được ghi cùng transaction với ledger.

**Yêu cầu cụ thể:**
- Bảng outbox phải lưu đủ payload cần thiết để tái tạo event kế toán (transaction ID, số tiền, currency, thời điểm) mà không cần join lại nhiều bảng khác lúc publish.
- Relay phải publish theo đúng thứ tự transaction được ghi (dùng cột thứ tự tăng dần/timestamp) để hệ thống kế toán nhận event đúng trình tự thời gian giao dịch.
- Trường hợp broker down kéo dài, outbox phải chịu được tồn đọng lớn (ví dụ hàng trăm nghìn record) mà không làm chậm transaction ghi giao dịch mới.
- Phải có cơ chế đối soát định kỳ: so khớp số lượng/checksum giao dịch trong ledger với số event đã publish thành công, phát hiện thiếu sót.
- Xử lý được multi-instance relay (scale nhiều instance để tăng throughput publish) mà không publish trùng cùng một outbox record.

---

## Đồng bộ dữ liệu khách hàng ra CRM bên ngoài

**Repository:** `outbox-customer-data-crm-sync`

**Hệ thống:** Một app SaaS cần đồng bộ mọi thay đổi thông tin khách hàng (tạo mới, cập nhật) ra một CRM bên thứ ba qua API.

**Vai trò của flow:** Outbox lưu lại thay đổi cần đồng bộ ngay trong transaction cập nhật DB nội bộ, để relay job gọi API CRM (có thể chậm, có rate limit) một cách độc lập và bền vững.

**Yêu cầu cụ thể:**
- Mỗi thay đổi khách hàng tạo một outbox record chứa đủ diff cần gửi; nếu khách hàng bị sửa nhiều lần liên tiếp trước khi relay kịp gửi, có chiến lược rõ ràng: gửi từng thay đổi riêng hay coalesce thành 1 lần gửi trạng thái mới nhất.
- Relay phải tôn trọng rate limit của CRM API (ví dụ tối đa 10 request/giây) và tự điều tiết tốc độ đọc outbox theo đó, không bắn dồn gây bị CRM chặn.
- Khi CRM API trả lỗi 4xx do dữ liệu không hợp lệ (không phải lỗi tạm thời), record đó phải được đưa vào trạng thái "cần review thủ công" thay vì retry vô hạn.
- Idempotency: CRM phải nhận diện được record trùng nếu outbox bị publish lại (dùng external ID nhất quán khi gọi API upsert).
- Có cảnh báo khi outbox tồn đọng vượt ngưỡng thời gian (ví dụ dữ liệu 1 giờ chưa được đồng bộ ra CRM).

---

## Inventory service publish sự kiện thay đổi tồn kho cho nhiều consumer nội bộ

**Repository:** `outbox-inventory-change-event`

**Hệ thống:** Inventory service là nguồn sự thật (source of truth) về tồn kho, nhiều service khác (pricing, search index, dashboard) cần biết mỗi khi tồn kho thay đổi.

**Vai trò của flow:** Outbox pattern đảm bảo mọi thay đổi tồn kho ghi vào DB đều chắc chắn được phát ra event, kể cả khi service tồn kho bị restart/crash ngay sau khi ghi DB.

**Yêu cầu cụ thể:**
- Transaction cập nhật tồn kho (ví dụ trừ hàng khi có đơn) và ghi outbox event "InventoryChanged" phải cùng một transaction, kể cả khi cập nhật xảy ra hàng loạt (batch update nhiều SKU).
- Relay phải publish theo batch để tối ưu throughput khi có sự kiện lớn (flash sale), nhưng vẫn đảm bảo không bỏ sót record nào (transactional read + mark).
- Có cơ chế phát hiện outbox relay bị "đứng" (không tiến triển publish index) và tự động failover sang instance relay khác.
- Thiết kế schema outbox tính đến việc archive định kỳ (partition theo thời gian) để việc query "record chưa publish" luôn nhanh dù bảng lịch sử rất lớn.
- Consumer downstream (search index, pricing) phải xử lý được event đến trùng hoặc không đúng thứ tự tuyệt đối, dựa vào version/sequence number đi kèm mỗi event.
