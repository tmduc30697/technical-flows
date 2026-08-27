# Zero-downtime schema migration & dual-write flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống web (mạng xã hội, e-commerce, fintech, logistics, SaaS đa tenant) để luyện việc đổi schema/đổi field/tách bảng mà không downtime, xử lý dual-write giữa schema cũ và mới khi traffic vẫn chạy liên tục.

---

## Tách bảng `orders` thành `orders` + `order_items` không downtime (e-commerce)

**Repository:** `schema-migration-ecommerce-orders-split`

**Hệ thống:** Một sàn e-commerce có bảng `orders` cũ lưu items dưới dạng JSON trong 1 cột, cần tách sang bảng quan hệ `order_items` để hỗ trợ báo cáo/join hiệu quả hơn.

**Vai trò của flow:** Trong lúc traffic đặt hàng vẫn chạy 24/7, flow phải đảm bảo mọi order mới tạo được ghi cả vào cột JSON cũ và bảng `order_items` mới, đồng thời có job backfill order cũ, mà không làm gián đoạn checkout.

**Yêu cầu cụ thể:**
- Yêu cầu transaction tạo order dual-write phải wrap trong 1 transaction DB duy nhất (ghi `orders` + insert `order_items`) để không có trường hợp order được tạo nhưng thiếu order_items do crash giữa chừng.
- Mô tả cụ thể tình huống 2 request checkout đồng thời cho cùng 1 cart (double-submit do user bấm 2 lần) trong giai đoạn dual-write: yêu cầu idempotency key theo `cart_id` + `checkout_attempt_id` để chỉ 1 trong 2 request tạo order thành công, request kia phải nhận lại kết quả của request đầu (không tạo order trùng ở cả bảng cũ và mới).
- Job backfill order lịch sử phải chạy bằng cursor theo `order_id` tăng dần theo batch nhỏ (ví dụ 500 order/lần), có checkpoint để resume nếu job bị crash giữa chừng, không đọc lại từ đầu.
- Định nghĩa cơ chế kiểm tra tính nhất quán (reconciliation job) so sánh số lượng/tổng tiền giữa cột JSON cũ và bảng `order_items` mới cho một mẫu order ngẫu nhiên mỗi ngày, cảnh báo nếu phát hiện lệch.
- Quy định rõ điều kiện để an toàn drop cột JSON cũ: phải chạy dual-write ổn định (không lệch dữ liệu) trong một khoảng thời gian tối thiểu, và đã chuyển 100% luồng đọc sang bảng mới.

---

## Đổi đơn vị tiền tệ lưu trữ từ float sang integer (cents) trong hệ thống fintech

**Repository:** `schema-migration-fintech-currency-cents`

**Hệ thống:** Một app quản lý chi tiêu cá nhân đang lưu số tiền dạng `float` (dễ sai số làm tròn), cần migrate toàn bộ sang lưu `integer` (đơn vị cents) mà không làm sai lệch số dư của user nào trong lúc migrate.

**Vai trò của flow:** Flow dual-write phải đảm bảo mọi giao dịch mới trong lúc migrate được ghi đúng ở cả cột `amount_float` (cũ) và `amount_cents` (mới), với công thức chuyển đổi nhất quán, để không tạo ra sai số dồn tích ảnh hưởng đến số dư hiển thị cho user.

**Yêu cầu cụ thể:**
- Quy định công thức chuyển đổi cụ thể (ví dụ round-half-up ở đơn vị cents) và bắt buộc áp dụng đồng nhất ở cả chiều ghi mới và chiều backfill dữ liệu cũ, để tránh 2 cách round khác nhau tạo lệch số dư giữa 2 cột.
- Mô tả race condition: giao dịch A (ghi tiền vào) và giao dịch B (rút tiền ra) xảy ra gần như đồng thời trên cùng 1 tài khoản trong giai đoạn dual-write — yêu cầu cả 2 cột `amount_float` và `amount_cents` phải được cập nhật trong cùng transaction với cùng 1 lock trên dòng số dư, không cho phép trạng thái chỉ 1 cột được update.
- Job backfill lịch sử giao dịch phải tính lại `amount_cents` từ `amount_float` gốc (không phải từ số dư đã tính toán ngược), và phải chạy khi hệ thống ở mức traffic thấp theo batch, có cơ chế tạm dừng nếu phát hiện tỷ lệ lệch số dư vượt ngưỡng.
- Có bước validate: sau khi backfill xong mỗi batch, tổng số dư tính theo cột cũ và cột mới của từng user phải khớp trong sai số cho phép (ví dụ dưới 1 cent do rounding), nếu lệch phải chặn không cho tiếp tục sang batch kế và cảnh báo ngay.
- Định nghĩa rollback plan cụ thể: nếu phát hiện lỗi nghiêm trọng sau khi đã chuyển đọc sang cột mới, phải có cách quay lại đọc cột cũ ngay lập tức (feature flag) mà không cần deploy lại code.

---

## Đổi từ multi-tenant chung schema sang schema-per-tenant trong SaaS B2B

**Repository:** `schema-migration-b2b-saas-schema-per-tenant`

**Hệ thống:** Một SaaS quản lý nhân sự B2B đang dùng 1 schema chung với cột `tenant_id` để phân biệt dữ liệu các công ty khách hàng, cần chuyển một số tenant lớn sang schema riêng để cải thiện performance.

**Vai trò của flow:** Với mỗi tenant đang migrate, flow phải đảm bảo request ghi dữ liệu (thêm nhân viên, sửa lương...) trong lúc chuyển đổi vẫn đi đúng vào schema đang là "nguồn sự thật" hiện tại của tenant đó, tránh việc 1 request ghi vào schema cũ sau khi cutover đã hoàn tất.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế "routing table" theo `tenant_id` xác định tenant đang ở schema nào (cũ/đang migrate/đã chuyển xong), và mọi request phải tra bảng này trước khi ghi — không hardcode schema trong code app.
- Mô tả cụ thể tình huống cutover: ngay tại thời điểm chuyển routing của 1 tenant từ schema cũ sang schema mới, có request đang giữa transaction ghi vào schema cũ — yêu cầu cơ chế "khóa ghi tạm thời" (maintenance window ngắn, vài giây) cho riêng tenant đó trong lúc cutover, các request khác của tenant khác không bị ảnh hưởng.
- Trong giai đoạn dual-write trước cutover, mọi write cho tenant đang migrate phải ghi cả 2 schema trong cùng 1 transaction phân tán (hoặc pattern outbox/saga nếu 2 schema ở 2 connection khác nhau), có xử lý rõ khi 1 trong 2 bên ghi thất bại (retry hoặc rollback toàn bộ, không để 1 bên có 1 bên không).
- Yêu cầu đọc dữ liệu trong giai đoạn dual-write luôn đọc từ schema cũ (nguồn sự thật) cho tới khi cutover, để tránh đọc dữ liệu chưa backfill đầy đủ ở schema mới.
- Có checklist rollback: nếu sau cutover phát hiện lỗi dữ liệu ở schema mới, phải trả routing của tenant đó về schema cũ trong vòng vài phút mà không mất giao dịch nào ghi vào schema mới trong khoảng thời gian đã cutover.

---

## Tách comment dạng nested JSON trong bảng posts sang bảng comment quan hệ riêng (mạng xã hội)

**Repository:** `schema-migration-social-posts-comments-split`

**Hệ thống:** Một mạng xã hội có bảng `posts` lưu comment dạng nested/JSON ngay trong 1 cột của bài viết, cần tách sang bảng `comments` quan hệ riêng để hỗ trợ tính năng mới (trả lời lồng nhiều cấp, like riêng từng comment, phân trang hiệu quả), trong khi traffic đăng bài/bình luận vẫn chạy liên tục 24/7.

**Vai trò của flow:** Flow dual-write phải đảm bảo mọi comment mới được ghi cả vào JSON cũ và bảng `comments` mới, có job backfill comment lịch sử, mà không làm rớt comment hay đổi thứ tự hiển thị trong lúc traffic bình luận vẫn cao liên tục.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: 2 người dùng cùng bình luận vào 1 bài viết đang hot gần như đồng thời (chênh nhau vài trăm mili giây) trong giai đoạn dual-write — việc ghi vào cột JSON cũ (thường phải đọc JSON hiện tại, append, rồi ghi lại cả mảng) không được dùng kiểu đọc-sửa-ghi không atomic vì 2 request đọc cùng JSON gốc rồi ghi đè nhau sẽ làm mất 1 trong 2 comment; yêu cầu mô tả cụ thể cách xử lý (lock dòng post trong lúc append JSON, hoặc chuyển hẳn logic ghi JSON sang dạng append atomic ở tầng lưu trữ), trong khi bảng `comments` mới insert bình thường không gặp vấn đề này vì mỗi comment là 1 dòng riêng.
- Trong giai đoạn đọc vẫn lấy từ JSON cũ (nguồn sự thật) nhưng ghi đã chạy song song cả 2 nơi, phải đảm bảo thứ tự comment trong JSON và thứ tự theo timestamp trong bảng `comments` khớp nhau tuyệt đối; yêu cầu cơ chế kiểm tra để phát hiện lệch thứ tự (ví dụ do 1 trong 2 lần ghi bị chậm hơn lần kia dưới tải cao) trước khi cho phép chuyển đọc sang bảng mới.
- Mô tả cụ thể race với reply lồng nhiều cấp: 1 comment cha vừa được tạo và ngay lập tức có người trả lời comment đó (comment con) trước khi thao tác ghi JSON cho comment cha kịp hoàn tất ở phía JSON cũ (do append JSON có độ trễ cao hơn insert bảng quan hệ) — quy định rõ cách xử lý để comment con không bị ghi vào JSON cũ ở vị trí sai cấu trúc lồng, ví dụ chờ xác nhận comment cha đã ghi xong JSON trước khi cho phép reply được xử lý, hoặc retry ghi JSON theo đúng thứ tự cha rồi mới tới con.
- Job backfill comment lịch sử phải chạy theo batch ưu tiên các bài viết có traffic bình luận đang hoạt động thấp trước, có checkpoint để resume nếu crash giữa chừng, và phải bỏ qua/log riêng các bài viết có comment JSON bị lỗi cấu trúc (dữ liệu cũ có thể không đồng nhất do đã đổi format nhiều lần trước đây) thay vì làm crash toàn bộ job.
- Có job reconciliation so sánh số lượng comment và nội dung mẫu giữa JSON và bảng mới cho các bài viết có traffic cao mỗi ngày, và chỉ được drop cột JSON cũ sau khi 100% traffic đọc đã chuyển sang bảng mới và không phát hiện lệch trong khoảng thời gian tối thiểu quy định.

---

## Đổi cấu trúc lưu trạng thái vận đơn từ 1 cột status sang bảng lịch sử trạng thái (logistics)

**Repository:** `schema-migration-logistics-shipment-status-history`

**Hệ thống:** Một nền tảng logistics tổng hợp trạng thái vận đơn từ nhiều đối tác vận chuyển khác nhau, hiện lưu trạng thái vận đơn ở 1 cột `status` đơn trên bảng `shipments`, cần chuyển sang bảng `shipment_status_history` để lưu đầy đủ lịch sử các mốc trạng thái phục vụ tra cứu hành trình chi tiết, trong khi đơn hàng vẫn đang di chuyển và nhận cập nhật trạng thái liên tục qua nhiều đối tác vận chuyển.

**Vai trò của flow:** Flow dual-write phải ghi mọi cập nhật trạng thái mới vào cả cột `status` cũ và bảng lịch sử mới, xử lý đúng khi các cập nhật từ nhiều đối tác vận chuyển cho cùng 1 vận đơn đến gần như đồng thời hoặc không đúng thứ tự thời gian thực.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: 2 cập nhật trạng thái cho cùng 1 vận đơn đến gần như đồng thời từ 2 nguồn khác nhau (webhook từ đối tác chặng đầu báo "đã giao cho đối tác chặng 2" và polling job tự phát hiện trạng thái "đang trung chuyển" cùng lúc) — yêu cầu ghi cột `status` cũ (chỉ giữ 1 giá trị cuối) và bảng lịch sử mới (insert thêm 1 dòng) trong cùng 1 transaction, xác định trạng thái nào là "mới nhất thật sự" dựa trên timestamp gốc của sự kiện (không phải thời điểm hệ thống nhận được), tránh cột `status` cũ bị ghi giá trị cũ hơn đè lên giá trị mới hơn do thứ tự xử lý ở tầng ứng dụng không khớp thứ tự thời gian thực.
- Trạng thái đến trễ/không đúng thứ tự là bình thường trong logistics do độ trễ khác nhau giữa các đối tác vận chuyển: bảng lịch sử mới phải chấp nhận insert mọi sự kiện theo đúng trình tự nhận được (không từ chối hay ghi đè), nhưng cột `status` cũ chỉ phản ánh trạng thái ứng với timestamp lớn nhất trong các sự kiện đã nhận — yêu cầu test cụ thể giả lập 3 cập nhật đến không đúng thứ tự thời gian, xác nhận `status` cuối cùng đúng và bảng lịch sử vẫn lưu đủ cả 3 sự kiện theo đúng trình tự thực.
- Xử lý webhook trùng lặp do đối tác vận chuyển retry (vì không nhận được ack kịp thời): idempotent theo mã sự kiện/mã trạng thái từ đối tác để không insert trùng dòng lịch sử khi cùng 1 webhook đến nhiều lần, trong khi vẫn đảm bảo webhook hợp lệ đầu tiên luôn được ghi dù đến đúng lúc đang có traffic cập nhật cao từ các vận đơn khác.
- Job backfill lịch sử cho các vận đơn đang active (chưa giao xong) phải chạy mà không làm mất các cập nhật trạng thái mới đến ngay trong lúc job đang backfill đúng vận đơn đó — quy định cơ chế khóa ngắn hoặc kiểm tra lại trạng thái sau khi backfill để không bị cập nhật real-time đến giữa chừng ghi đè mất.
- Chỉ chuyển toàn bộ luồng đọc (tra cứu hành trình cho khách, dashboard vận hành) sang bảng lịch sử mới sau khi đã backfill xong toàn bộ vận đơn đang active và dual-write chạy ổn định không lệch trong khoảng thời gian tối thiểu quy định, đặc biệt phải test kỹ các vận đơn có tần suất cập nhật trạng thái cao (đơn liên tỉnh qua nhiều chặng trung chuyển) trước khi áp dụng cho toàn hệ thống.
