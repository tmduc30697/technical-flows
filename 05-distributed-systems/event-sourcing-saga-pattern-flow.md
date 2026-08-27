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

---

## Chi trả tiền cho seller trên nền tảng marketplace sau khi đơn hàng hoàn tất

**Repository:** `saga-marketplace-seller-payout`

**Hệ thống:** Sàn marketplace có hàng nghìn seller, sau khi đơn hàng hoàn tất (khách nhận hàng, hết thời gian khiếu nại) hệ thống phải tính toán và chuyển tiền cho seller qua đối tác thanh toán bên ngoài.

**Vai trò của flow:** Saga điều phối chuỗi bước tính hoa hồng/phí, trừ vào số dư seller, và gọi đối tác thanh toán để chuyển tiền thực — đảm bảo nếu bước chuyển tiền thất bại giữa chừng, phần đã trừ được compensate đúng, không làm seller mất tiền oan hoặc bị trừ hai lần.

**Yêu cầu cụ thể:**
- Nếu bước gọi đối tác thanh toán trả về timeout (không rõ tiền đã chuyển hay chưa) sau khi số dư seller đã bị trừ trong hệ thống nội bộ, saga không được tự động compensate ngay (hoàn lại số dư) mà phải verify trạng thái thực tế qua API đối chiếu/callback của đối tác trước, tránh vừa mất tiền chuyển thật vừa hoàn số dư ảo dẫn tới double-pay.
- Compensate phần đã trừ (hoa hồng, phí) chỉ được thực hiện khi chắc chắn bước chuyển tiền chưa xảy ra hoặc đã thất bại dứt khoát được đối tác xác nhận; nếu chuyển tiền đã thành công nhưng saga crash trước khi ghi nhận, resume saga phải phát hiện qua truy vấn đối tác thay vì chạy lại từ đầu gây chuyển tiền trùng.
- Batch payout xử lý hàng nghìn seller cùng lúc — một seller lỗi (tài khoản ngân hàng không hợp lệ) không được làm dừng/rollback toàn bộ batch của các seller khác; saga cho mỗi seller phải độc lập và tiếp tục song song, chỉ đánh dấu payout của seller lỗi để retry riêng.
- Retry payout cho một seller sau lỗi tạm thời (đối tác timeout) phải idempotent ở phía đối tác thanh toán (dùng transaction reference cố định), tránh trường hợp retry vô tình tạo ra hai lệnh chuyển tiền thực cho cùng một khoản payout.
- Toàn bộ event của saga payout (tính hoa hồng, trừ số dư, gọi đối tác, xác nhận/compensate) phải lưu thành event log truy vấn được theo từng seller/đơn hàng, phục vụ đối soát tài chính và giải trình khi seller khiếu nại về số tiền nhận được.

---

## Điều phối vòng đời một chuyến đi ride-hailing qua nhiều bước

**Repository:** `saga-ride-hailing-trip-lifecycle`

**Hệ thống:** App gọi xe điều phối một chuyến đi qua chuỗi bước: tìm tài xế phù hợp → tài xế nhận cuốc → tài xế đón khách → hoàn thành chuyến đi → thanh toán, mỗi bước xảy ra ở thiết bị khách/tài xế khác nhau và có thể mất kết nối bất kỳ lúc nào.

**Vai trò của flow:** Saga theo dõi và điều phối trạng thái chuyến đi xuyên các service (matching, trip, payment), xử lý hủy/mất kết nối giữa chừng ở từng bước và đưa chuyến đi về trạng thái nhất quán thay vì kẹt lửng lơ.

**Yêu cầu cụ thể:**
- Nếu tài xế đã "nhận cuốc" nhưng sau đó tự hủy trước khi đón khách, saga phải compensate bằng cách trả chuyến đi về trạng thái "tìm tài xế" và loại trừ tài xế vừa hủy khỏi vòng matching tiếp theo cho cùng chuyến, tránh matching lặp lại ngay với tài xế vừa từ chối.
- Khi app khách mất kết nối đúng lúc tài xế đang trên đường đón (giữa bước "nhận cuốc" và "đón khách"), saga không được tự hủy chuyến ngay lập tức — cần cửa sổ chờ hợp lý để khách reconnect, vì hủy sai có thể khiến tài xế đã di chuyển gần tới nơi mất công vô ích và ảnh hưởng đánh giá.
- Race condition khi khách hủy chuyến đúng lúc tài xế bấm "đã đón khách" gửi lên gần như đồng thời — hai sự kiện đến từ hai thiết bị khác nhau qua network độc lập, saga phải có luật ưu tiên rõ ràng (ví dụ trạng thái nào server nhận trước theo thời điểm xử lý được coi là hợp lệ) và bên thua phải nhận thông báo đúng trạng thái thực tế, không để hai app hiển thị hai trạng thái mâu thuẫn.
- Bước thanh toán ở cuối saga có thể thất bại (thẻ bị từ chối) sau khi chuyến đã hoàn thành thực tế — saga không được compensate bằng cách "hủy chuyến" vì chuyến đã xảy ra không thể hủy ngược, mà phải tách thành trạng thái "hoàn thành, chờ xử lý thanh toán" với luồng retry/thu nợ riêng, đồng thời vẫn ghi nhận thu nhập tạm cho tài xế.
- Saga phải resume đúng bước khi service điều phối trip bị restart giữa lúc chuyến đang chạy (ví dụ deploy giữa lúc có hàng nghìn chuyến đang diễn ra) — trạng thái từng chuyến phải persist đủ để không gửi lại thông báo "đã tìm thấy tài xế" cho chuyến đã qua bước đó từ trước, gây nhiễu loạn app khách/tài xế.
