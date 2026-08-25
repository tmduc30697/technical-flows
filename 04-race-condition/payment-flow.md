# Payment flow (xử lý giao dịch + callback + đối soát) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống có xử lý thanh toán (e-commerce, subscription SaaS, ví điện tử, cổng thanh toán tổng hợp, thương mại xuyên biên giới) để luyện xử lý giao dịch thanh toán, callback/webhook không đảm bảo thứ tự hoặc bị gửi trùng, và đối soát dữ liệu với bên thứ ba.

---

## Thanh toán qua cổng bên thứ ba với webhook callback không đảm bảo thứ tự

**Repository:** `payment-gateway-webhook-out-of-order`

**Hệ thống:** Một sàn e-commerce tích hợp cổng thanh toán bên thứ ba (kiểu Stripe/VNPay), nhận callback (webhook) báo kết quả giao dịch sau khi khách thanh toán.

**Vai trò của flow:** Flow xử lý callback phải cập nhật đúng trạng thái đơn hàng dựa trên webhook nhận được, kể cả khi webhook bị gửi trùng nhiều lần hoặc đến không đúng thứ tự thời gian thực tế xảy ra.

**Yêu cầu cụ thể:**
- Webhook phải được xử lý idempotent theo `transaction_id` của cổng thanh toán: nếu nhận cùng 1 webhook 3 lần (do cổng thanh toán retry vì response chậm), chỉ áp dụng thay đổi trạng thái đúng 1 lần, 2 lần sau chỉ trả 200 OK mà không làm gì thêm.
- Mô tả cụ thể tình huống thứ tự đảo lộn: webhook "payment_success" đến sau webhook "payment_refunded" của cùng giao dịch (do độ trễ mạng khác nhau) — yêu cầu dùng timestamp hoặc sequence number từ phía cổng thanh toán để xác định thứ tự thật, không dùng thời điểm hệ thống nhận được webhook để quyết định trạng thái cuối cùng.
- Khi webhook báo thành công đến gần như đồng thời với việc khách bấm "Hủy đơn" ở app, quy định rõ ai thắng: nếu tiền đã thực sự được trừ ở cổng thanh toán (webhook success là nguồn sự thật), đơn hàng không được hủy mà phải chuyển sang luồng hoàn tiền, không đơn giản đảo trạng thái theo request đến sau.
- Nếu server nội bộ bị down đúng lúc cổng thanh toán gửi webhook, phải đảm bảo cổng thanh toán sẽ retry theo cơ chế của họ (hoặc tự chủ động polling trạng thái giao dịch định kỳ như phương án dự phòng) — không để đơn hàng bị "treo" vĩnh viễn ở trạng thái "đang xử lý" chỉ vì lỡ mất đúng 1 webhook.
- Có job đối soát hàng ngày so sánh danh sách giao dịch thành công ghi nhận nội bộ với báo cáo từ cổng thanh toán, phát hiện và cảnh báo các giao dịch lệch (có ở 1 bên nhưng không có ở bên kia).

---

## Cổng thanh toán tổng hợp (payment orchestration) định tuyến qua nhiều nhà cung cấp

**Repository:** `payment-orchestration-multi-provider-routing`

**Hệ thống:** Một hệ thống trung gian định tuyến giao dịch thanh toán qua nhiều nhà cung cấp khác nhau (theo tỷ lệ, theo phí, theo tình trạng sẵn sàng của từng nhà cung cấp) để tối ưu tỷ lệ thành công.

**Vai trò của flow:** Flow phải chọn đúng 1 nhà cung cấp cho mỗi giao dịch, xử lý callback trả về từ nhiều nhà cung cấp có format khác nhau, và không để 1 giao dịch bị xử lý bởi 2 nhà cung cấp cùng lúc do lỗi retry.

**Yêu cầu cụ thể:**
- Khi request thanh toán đầu tiên gửi tới nhà cung cấp A bị timeout (không rõ đã xử lý hay chưa ở phía A), hệ thống không được tự động gửi luôn request y hệt tới nhà cung cấp B mà không kiểm tra trước — phải có bước xác minh trạng thái với A (query status) hoặc đợi đủ thời gian timeout xác định trước khi coi là thất bại và chuyển hướng qua B, tránh giao dịch bị xử lý (charge tiền khách) ở cả A và B.
- Mỗi giao dịch chỉ được gán đúng 1 nhà cung cấp xử lý tại một thời điểm — dùng cơ chế lock/trạng thái atomic (ví dụ `status = routing` → `status = sent_to_provider_A`) để tránh 2 worker xử lý hàng đợi routing cùng lúc gán nhầm 1 giao dịch cho 2 nhà cung cấp khác nhau.
- Callback từ các nhà cung cấp khác nhau có format khác nhau nhưng phải được chuẩn hóa về cùng 1 mô hình trạng thái nội bộ trước khi ghi nhận, và mọi callback phải map lại đúng giao dịch gốc qua mã tham chiếu đã gửi kèm lúc khởi tạo (không dựa vào số tiền + thời gian để đoán, dễ nhầm khi trùng giá trị).
- Mô tả cụ thể race giữa callback thật từ nhà cung cấp A báo thất bại và cơ chế failover tự động của hệ thống đã quyết định chuyển sang B do quá thời gian chờ — nếu callback A đến ngay sau khi B đã bắt đầu xử lý, phải đảm bảo giao dịch không bị double-charge (chỉ 1 trong 2 được coi là hợp lệ, phải hủy/hoàn tiền phía còn lại nếu cả 2 vô tình đều thành công).
- Có dashboard theo dõi tỷ lệ thành công/lỗi theo từng nhà cung cấp real-time để tự động điều chỉnh tỷ lệ routing, và log đầy đủ lịch sử routing của mỗi giao dịch (đã thử nhà cung cấp nào, theo thứ tự nào, kết quả gì) phục vụ điều tra khi có khiếu nại double-charge.

---

## Thanh toán thương mại xuyên biên giới với chuyển đổi tiền tệ và callback đa múi giờ

**Repository:** `payment-cross-border-multi-timezone-callback`

**Hệ thống:** Một sàn thương mại xuyên biên giới cho phép khách thanh toán bằng tiền tệ nội địa, hệ thống tự chuyển đổi sang tiền tệ của seller, callback báo kết quả từ đối tác thanh toán quốc tế có thể đến sau nhiều giờ.

**Vai trò của flow:** Flow phải khóa đúng tỷ giá tại thời điểm giao dịch, xử lý callback đến rất trễ (đôi khi hơn 24 giờ) mà không làm sai lệch số tiền do tỷ giá đã biến động, và đối soát được số tiền 2 loại tiền tệ.

**Yêu cầu cụ thể:**
- Tại thời điểm khách bấm thanh toán, tỷ giá quy đổi phải được "khóa" (snapshot) và lưu lại cùng giao dịch — callback xử lý kết quả sau đó (dù đến trễ bao lâu) phải dùng đúng tỷ giá đã khóa này để tính số tiền cuối cùng, không lấy tỷ giá hiện tại tại thời điểm callback đến.
- Mô tả cụ thể: callback báo giao dịch thành công đến sau 20 giờ, trong khoảng thời gian đó tỷ giá đã biến động 2%; hệ thống ghi nhận doanh thu cho seller phải dùng đúng tỷ giá đã khóa lúc khách thanh toán, không phải tỷ giá tại lúc nhận callback — có test cụ thể xác nhận số tiền quy đổi khớp với tỷ giá khóa ban đầu.
- Nếu callback báo thất bại đến sau khi hệ thống đã tạm ghi nhận đơn hàng ở trạng thái "đã thanh toán" (do quá lâu chưa nhận được callback nên có luồng dự phòng tự chuyển trạng thái theo polling), phải có cơ chế đảo ngược đúng: hủy đơn, hoàn tiền theo đúng tỷ giá đã khóa ban đầu (không phải tỷ giá hiện tại lúc hoàn tiền).
- Do đối tác thanh toán quốc tế ở múi giờ khác, các mốc "theo ngày" trong đối soát (ví dụ báo cáo cuối ngày) phải được chuẩn hóa về 1 múi giờ tham chiếu duy nhất (ví dụ UTC) khi so khớp dữ liệu giữa 2 hệ thống, tránh lệch báo cáo do 1 bên tính theo giờ địa phương.
- Có luồng đối soát riêng theo từng loại tiền tệ (không gộp chung), so khớp cả số tiền gốc (tiền khách trả) và số tiền quy đổi (tiền seller nhận), phát hiện các giao dịch có sai lệch tỷ giá bất thường để điều tra riêng.
