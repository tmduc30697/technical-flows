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

---

## Job tính phí renewal tự động chạy trùng lúc khách tự đổi gói/hủy gói trên subscription SaaS

**Repository:** `payment-saas-subscription-renewal-race`

**Hệ thống:** Một SaaS B2B tính phí theo subscription hàng tháng, có job billing tự động chạy vào đúng ngày renewal của mỗi khách để charge thẻ và gia hạn, khách hàng cũng có thể tự đổi gói hoặc hủy gói qua UI vào bất kỳ thời điểm nào.

**Vai trò của flow:** Flow phải xử lý đúng khi job billing tự động renewal và hành động thủ công của khách (đổi gói/hủy) chạm vào cùng 1 subscription gần như đồng thời, tránh charge sai gói hoặc charge sau khi khách đã hủy.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: job billing bắt đầu xử lý renewal cho subscription S lúc 00:00:05 (đọc gói hiện tại là "Pro"), đúng lúc khách bấm hủy subscription lúc 00:00:06, trước khi job kịp charge thẻ — yêu cầu job phải lock (hoặc dùng version check) trên record subscription trước khi charge, nếu phát hiện trạng thái đã đổi so với lúc đọc ban đầu (đã bị hủy) thì dừng ngay lập tức, không charge, không tạo invoice mới.
- Mô tả race khi khách đổi gói (downgrade từ Pro sang Basic) đúng lúc job renewal đang giữa quá trình charge theo gói Pro (đã gọi tới cổng thanh toán, đang chờ phản hồi) — quy định rõ lần renewal này vẫn hoàn tất theo gói Pro với giá đã khóa tại thời điểm job bắt đầu xử lý, việc đổi gói mới chỉ có hiệu lực từ chu kỳ kế tiếp; không được hủy giữa chừng giao dịch đang gọi cổng thanh toán vì có thể phía cổng đã charge thành công dù hệ thống nội bộ chưa kịp ghi nhận.
- Mô tả cụ thể: khách bấm hủy subscription ngay 5 giây sau khi job billing đã charge thành công chu kỳ mới — quy định rõ chính sách hoàn tiền theo tỷ lệ hoặc không hoàn cho phần thời gian còn lại của chu kỳ, và đảm bảo trạng thái subscription cuối cùng nhất quán, không rơi vào tình trạng vừa "đã charge chu kỳ mới" vừa "đã hủy ngay lập tức" gây mâu thuẫn khi hiển thị cho khách.
- Job renewal chạy hàng loạt cho nhiều subscription trong cùng 1 batch theo ngày: đảm bảo mỗi subscription được xử lý độc lập với 1 lock/version check riêng, lỗi hoặc conflict xảy ra ở 1 subscription (do khách đang thao tác trùng lúc) không được làm chậm hoặc ảnh hưởng tới việc xử lý các subscription khác trong cùng batch.
- Đảm bảo idempotency của job renewal: nếu job bị crash giữa chừng sau khi đã charge thành công ở cổng thanh toán nhưng chưa kịp cập nhật trạng thái subscription (ví dụ crash ngay sau khi gọi cổng thanh toán, trước khi ghi nhận kết quả), lần chạy lại job cho cùng subscription trong cùng chu kỳ không được charge lần 2 — phải kiểm tra trạng thái/idempotency key trước khi gọi cổng thanh toán lại.

---

## Nạp tiền và thanh toán từ ví điện tử chạm cùng 1 số dư gần như đồng thời

**Repository:** `payment-ewallet-balance-concurrent-topup-payment`

**Hệ thống:** Một ví điện tử cho phép người dùng nạp tiền từ ngân hàng liên kết vào ví và dùng số dư ví để thanh toán tại nhiều điểm chấp nhận thanh toán khác nhau (trong app, tại các merchant đối tác).

**Vai trò của flow:** Flow phải đảm bảo số dư ví được cộng/trừ chính xác khi lệnh nạp tiền và lệnh thanh toán chạm vào cùng 1 số dư gần như đồng thời, không để số dư bị âm hoặc bị tính sai do 2 luồng cập nhật race nhau trên cùng 1 số dư.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: khách có số dư 100.000đ, cùng lúc có 1 lệnh nạp tiền 50.000đ đang được xử lý (callback từ ngân hàng vừa báo thành công) và 1 lệnh thanh toán 120.000đ tại merchant đang được xử lý — nếu 2 luồng đều đọc số dư 100.000đ trước khi tính toán theo kiểu đọc-tính-ghi không atomic, lệnh thanh toán có thể bị từ chối sai (100.000 < 120.000 dù thực tế sau khi nạp là 150.000 đủ trả) hoặc tệ hơn cả 2 ghi đè lẫn nhau gây sai số dư cuối cùng; yêu cầu dùng update nguyên tử có điều kiện, trừ tiền chỉ khi số dư đủ tại đúng thời điểm ghi, không dựa vào số dư đã đọc trước đó.
- Quy định rõ hệ thống không cam kết thứ tự "nạp trước hay trừ trước" theo thời điểm người dùng bấm, mà theo thứ tự thực sự được xử lý atomic tại tầng lưu trữ — nếu lệnh thanh toán được xử lý trước khi tiền nạp kịp cộng vào và số dư tại thời điểm đó không đủ, phải từ chối thanh toán rõ ràng ngay, không giữ pending chờ nạp xong rồi tự động thử lại ngầm mà không thông báo cho khách.
- Mô tả cụ thể: khách thực hiện 2 lệnh thanh toán khác nhau gần như đồng thời từ 2 thiết bị/session (ví dụ quét mã QR ở 2 merchant liên tiếp trong vài giây) trong khi số dư chỉ đủ cho 1 trong 2 lệnh — yêu cầu update nguyên tử trừ số dư có điều kiện đủ số dư để chỉ 1 giao dịch thành công, giao dịch còn lại bị từ chối ngay với lý do rõ ràng "số dư không đủ", không để cả 2 cùng trừ thành công gây số dư âm.
- Callback nạp tiền từ ngân hàng đến trễ hoặc bị gửi trùng (do retry) đúng lúc ví đang có nhiều giao dịch trừ tiền khác diễn ra: xử lý idempotent theo mã giao dịch nạp tiền để không cộng tiền 2 lần khi callback đến trùng, đồng thời đảm bảo việc cộng tiền vẫn atomic với các giao dịch trừ tiền đang chạy song song, không để có khoảng hở giữa lúc callback bắt đầu xử lý và lúc số dư thực sự được cộng.
- Có job định kỳ đối chiếu tổng số dư ví của từng khách với tổng lịch sử giao dịch (nạp trừ hoàn) để phát hiện lệch do lỗi race hiếm gặp lọt qua các cơ chế atomic, cảnh báo và tạm khóa giao dịch của tài khoản bị lệch để điều tra trước khi khách phát hiện và khiếu nại.
