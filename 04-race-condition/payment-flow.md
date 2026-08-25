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

## Gia hạn subscription tự động (recurring billing) cho SaaS

**Repository:** `payment-saas-recurring-billing`

**Hệ thống:** Một SaaS B2B tính phí theo tháng, tự động charge thẻ đã lưu của khách vào đúng ngày gia hạn mỗi tháng.

**Vai trò của flow:** Flow billing job phải xử lý charge tự động cho hàng nghìn khách hàng cùng lúc vào đầu tháng, xử lý đúng khi charge thất bại (thẻ hết hạn, không đủ tiền) và callback báo kết quả có thể đến trễ hoặc trùng.

**Yêu cầu cụ thể:**
- Job billing chạy theo batch hàng nghìn subscription, mỗi subscription phải có idempotency key riêng cho lần charge của tháng đó (ví dụ theo `subscription_id + billing_period`), để nếu job bị chạy lại (do lỗi/crash giữa batch) không charge khách 2 lần cho cùng 1 tháng.
- Mô tả cụ thể race: khách hàng tự nâng cấp gói (upgrade plan, thay đổi giá subscription) đúng lúc job billing tự động đang xử lý charge tháng đó theo giá cũ — quy định rõ transaction charge phải lock đúng bản ghi subscription và đọc giá tại thời điểm charge thực sự diễn ra (không dùng giá đã cache từ đầu job), tránh charge sai giá do đổi gói giữa lúc xử lý.
- Khi callback từ cổng thanh toán báo "charge thất bại do thẻ hết hạn" đến trễ (vài giờ sau khi job đã đánh dấu tạm "đang xử lý"), hệ thống phải chuyển đúng trạng thái subscription sang "cần cập nhật phương thức thanh toán" và gửi thông báo khách, không để subscription treo ở trạng thái mơ hồ.
- Nếu 2 callback báo kết quả khác nhau cho cùng 1 lần charge (ví dụ do cổng thanh toán gửi trùng nhưng nội dung bị cập nhật giữa 2 lần gửi — hiếm nhưng phải xử lý), quy định nguồn dữ liệu nào được coi là chính xác cuối cùng (ví dụ luôn ưu tiên trạng thái mới nhất theo timestamp từ cổng thanh toán, không phải theo thời điểm nhận).
- Có cơ chế retry charge thất bại theo lịch cụ thể (ví dụ thử lại sau 3 ngày, 7 ngày trước khi hủy subscription), và mỗi lần retry phải dùng idempotency key khác (theo attempt number) để phân biệt với lần charge gốc, tránh cổng thanh toán coi là request trùng và không xử lý.

---

## Ví điện tử nạp tiền qua nhiều phương thức (ngân hàng, thẻ, ví liên kết) cùng lúc

**Repository:** `payment-ewallet-multi-method-topup`

**Hệ thống:** Một ví điện tử cho phép người dùng nạp tiền từ nhiều nguồn (chuyển khoản ngân hàng, thẻ tín dụng, liên kết ví khác), mỗi nguồn có callback riêng với format và độ trễ khác nhau.

**Vai trò của flow:** Flow xử lý callback nạp tiền phải cộng đúng số dư ví bất kể nguồn nào, tránh cộng trùng khi callback bị gửi lại, và xử lý đúng khi 2 giao dịch nạp tiền từ 2 nguồn khác nhau của cùng 1 user đến gần như đồng thời.

**Yêu cầu cụ thể:**
- Mỗi giao dịch nạp tiền phải có mã tham chiếu duy nhất xuyên suốt từ lúc tạo lệnh nạp tới lúc nhận callback, và transaction cộng số dư phải check đã xử lý mã này chưa trước khi cộng (idempotency ở tầng ghi số dư, không chỉ ở tầng nhận webhook).
- Mô tả cụ thể: user nạp 100k qua ngân hàng và 200k qua thẻ gần như đồng thời, 2 callback đến gần nhau — cả 2 transaction cộng số dư đều phải `UPDATE balance = balance + amount` (cộng dồn atomic ở DB), không đọc số dư hiện tại rồi tính tổng ở application rồi ghi đè (dễ làm mất 1 trong 2 giao dịch nếu đọc-sửa-ghi không lock đúng dòng).
- Khi callback báo nạp tiền thành công nhưng đến sau khi user đã báo lỗi qua support "không thấy tiền vào ví" và support đã thao tác cộng tiền tay để xử lý tạm, hệ thống phải phát hiện được khả năng cộng trùng (1 lần tay + 1 lần tự động cho cùng mã tham chiếu) và có cảnh báo đối soát, không để mất dấu.
- Nếu 1 giao dịch nạp tiền bị callback báo "thất bại" sau khi trước đó đã có 1 callback báo "đang xử lý" (2 trạng thái khác nhau theo thời gian), quy định rõ trạng thái cuối cùng phải theo callback mới nhất theo timestamp thực từ nguồn thanh toán, và nếu số dư đã bị cộng tạm theo trạng thái "đang xử lý" (không nên xảy ra nhưng phải phòng vệ), phải có cơ chế trừ lại đúng số tiền khi biết chắc là thất bại.
- Có báo cáo đối soát cuối ngày theo từng phương thức nạp tiền (ngân hàng/thẻ/ví liên kết), so khớp tổng số tiền và số lượng giao dịch giữa hệ thống nội bộ và báo cáo của từng đối tác, cảnh báo rõ theo từng phương thức nếu lệch.

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
