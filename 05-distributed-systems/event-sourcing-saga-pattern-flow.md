# Event sourcing/Saga pattern flow (rollback/compensate xuyên nhiều service) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần điều phối transaction nghiệp vụ xuyên service — checkout e-commerce, đặt combo du lịch, payout marketplace, vòng đời chuyến đi ride-hailing, và thay đổi subscription — nhằm luyện thiết kế saga step/compensate, event log, và khả năng resume sau crash trong web app thực tế.

---

## Checkout e-commerce nhiều bước (payment + inventory + shipping)

**Repository:** `saga-ecommerce-checkout`

**Hệ thống:** Trang e-commerce với luồng đặt hàng chạm tới 3 service riêng: Payment, Inventory, Shipping.

**Vai trò của flow:** Saga orchestration điều phối toàn bộ transaction nghiệp vụ xuyên service, và compensate (hủy ngược) khi một bước giữa đường thất bại.

**Yêu cầu cụ thể:**
- Định nghĩa rõ từng bước saga (reserve inventory → charge payment → create shipment) và compensating action tương ứng cho mỗi bước (release inventory, refund payment, cancel shipment).
- Nếu bước charge payment thất bại sau khi đã reserve inventory thành công, phải tự động gọi compensate để release inventory, và toàn bộ order chuyển trạng thái "failed" chứ không kẹt ở trạng thái lửng lơ.
- Saga state (đang ở bước nào, đã compensate bước nào) phải được persist, để nếu orchestrator service crash giữa saga, có thể resume đúng chỗ khi restart.
- Xử lý được trường hợp compensate cũng thất bại (ví dụ refund payment lỗi) — phải có retry với backoff và escalate sang xử lý thủ công/alert nếu vượt số lần retry.
- Toàn bộ event của saga (bắt đầu, mỗi step, compensate) phải được lưu dưới dạng event log để có thể replay/audit lại toàn bộ hành trình của một order.

---

## Đặt combo du lịch (vé máy bay + khách sạn + thuê xe)

**Repository:** `saga-travel-combo-booking`

**Hệ thống:** Nền tảng đặt tour cho phép khách đặt đồng thời vé máy bay, khách sạn, xe thuê từ 3 nhà cung cấp/service khác nhau.

**Vai trò của flow:** Saga đảm bảo tính "tất cả hoặc không có gì" ở mức nghiệp vụ (không có 2-phase commit thật) — nếu một phần đặt thất bại, phải hủy các phần đã đặt thành công trước đó.

**Yêu cầu cụ thể:**
- Thứ tự đặt phải được thiết kế để phần dễ hủy/ít rủi ro nhất được đặt trước (ví dụ giữ chỗ tạm thời trước khi charge tiền thật).
- Mỗi service cung cấp (airline, hotel, car) có API riêng với SLA/latency khác nhau — saga phải có timeout riêng cho từng bước và coi timeout là "thất bại cần compensate".
- Compensate hủy khách sạn/vé máy bay có thể mất phí hủy (cancellation fee) — phải log và tính đúng số tiền hoàn lại cho khách, không hoàn nhầm 100% khi có phí hủy.
- Cho khách xem được trạng thái real-time của từng phần đặt (đang xử lý/thành công/đã hủy) trong lúc saga đang chạy.
- Có cơ chế phát hiện saga "kẹt" quá lâu ở một bước (worker xử lý bước đó chết) và tự động resume hoặc alert vận hành.

---

## Nâng/hạ cấp gói subscription xuyên nhiều service (billing + entitlement + notification)

**Repository:** `saga-subscription-plan-change`

**Hệ thống:** SaaS B2B cho phép khách hàng nâng cấp/hạ cấp plan, việc này phải đồng bộ giữa billing (tính tiền), entitlement (mở/khóa tính năng), và notification (báo khách).

**Vai trò của flow:** Saga đảm bảo billing và entitlement luôn khớp nhau — không có trường hợp khách bị charge tiền plan mới nhưng chưa được mở tính năng, hoặc ngược lại.

**Yêu cầu cụ thể:**
- Thứ tự saga: charge/pro-rate billing trước, chỉ mở entitlement sau khi billing xác nhận thành công; nếu billing thất bại, không chạm tới entitlement.
- Nếu entitlement service down đúng lúc cần cập nhật sau khi billing đã thành công, saga phải retry với backoff trong một khoảng thời gian giới hạn, và nếu vẫn thất bại phải compensate (hoàn tiền phần chênh lệch) và giữ plan cũ.
- Downgrade plan phải xử lý được trường hợp khách đang dùng tính năng vượt hạn mức plan mới (ví dụ vượt số user) — quyết định rõ: chặn downgrade hay downgrade kèm cảnh báo.
- Notification chỉ được gửi sau khi cả billing và entitlement đã ổn định (không gửi email "đã nâng cấp thành công" rồi sau đó saga phải rollback).
- Toàn bộ lịch sử thay đổi plan của một khách phải truy vấn lại được (ai đổi plan gì, khi nào, kết quả ra sao) để phục vụ hỗ trợ khách hàng.
