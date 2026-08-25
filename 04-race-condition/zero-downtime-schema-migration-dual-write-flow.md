# Zero-downtime schema migration & dual-write flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống web (mạng xã hội, e-commerce, fintech, logistics, SaaS đa tenant) để luyện việc đổi schema/đổi field/tách bảng mà không downtime, xử lý dual-write giữa schema cũ và mới khi traffic vẫn chạy liên tục.

---

## Đổi kiểu lưu số điện thoại từ 1 field sang bảng riêng (mạng xã hội)

**Repository:** `schema-migration-social-phone-table-split`

**Hệ thống:** Một mạng xã hội cho phép user thêm nhiều số điện thoại (hiện lưu 1 field `phone` trong bảng `users`, cần chuyển sang bảng `user_phones` hỗ trợ nhiều số).

**Vai trò của flow:** Trong giai đoạn chuyển đổi, mọi request update số điện thoại phải ghi đồng thời (dual-write) vào cả field cũ và bảng mới để hai luồng đọc (code cũ chưa deploy xong, code mới đã deploy) đều thấy dữ liệu đúng, không mất update nào.

**Yêu cầu cụ thể:**
- Thiết kế thứ tự dual-write cụ thể: ghi vào bảng mới `user_phones` trước, sau đó update field cũ `users.phone`, và giải thích vì sao thứ tự này an toàn hơn nếu có crash giữa 2 bước (bảng mới là nguồn sự thật tạm thời).
- Mô tả race condition cụ thể: request A update số điện thoại qua code cũ (chỉ ghi field `phone`) đồng thời với request B update qua code mới (dual-write cả 2 nơi) — chỉ ra cách đảm bảo instance nào deploy code cũ trong lúc rollout cũng không làm dữ liệu bảng mới bị lệch (ví dụ dùng trigger DB đồng bộ 2 chiều tạm thời trong giai đoạn transition).
- Có bước backfill dữ liệu cũ sang bảng mới cho toàn bộ user hiện có, chạy theo batch, không lock bảng `users` quá lâu (đo và giới hạn thời gian mỗi batch, ví dụ dưới 200ms).
- Định nghĩa rõ 3 giai đoạn rollout: (1) dual-write + đọc từ field cũ, (2) dual-write + đọc từ bảng mới (đã backfill xong), (3) chỉ ghi/đọc bảng mới, xóa field cũ — mỗi giai đoạn phải chạy ổn định một thời gian trước khi sang giai đoạn kế.
- Xử lý case update đồng thời 2 request khác nhau cho cùng 1 user trong giai đoạn dual-write: đảm bảo không có tình trạng bảng mới có giá trị A còn field cũ có giá trị B (write ra thứ tự khác nhau do 2 request chạy song song không lock).

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

## Thêm cột `status` có logic mới thay thế cho 3 cột boolean cũ trong hệ thống logistics

**Repository:** `schema-migration-logistics-status-column`

**Hệ thống:** Một hệ thống quản lý vận đơn (logistics) đang lưu trạng thái đơn hàng bằng 3 cột boolean riêng (`is_picked`, `is_shipped`, `is_delivered`), cần chuyển sang 1 cột `status` enum để dễ mở rộng thêm trạng thái mới.

**Vai trò của flow:** Trong lúc migrate, mọi transition trạng thái đơn hàng (worker quét mã khi lấy hàng, khi giao hàng...) phải ghi nhất quán cả 3 cột boolean cũ và cột `status` mới, đảm bảo 2 cách biểu diễn luôn tương đương nhau dù có bao nhiêu worker cùng cập nhật đơn hàng đồng thời.

**Yêu cầu cụ thể:**
- Định nghĩa bảng mapping tường minh giữa tổ hợp 3 boolean cũ và giá trị enum mới (bao gồm cả các tổ hợp không hợp lệ về mặt logic hiện tại, ví dụ `is_delivered=true` nhưng `is_shipped=false`) và quy tắc xử lý khi gặp dữ liệu cũ không hợp lệ lúc backfill.
- Mô tả race condition cụ thể: 2 worker quét mã cùng lúc cho cùng 1 đơn hàng ở 2 trạm khác nhau (một quét "shipped", một quét "delivered" do lỗi quy trình) — yêu cầu transaction update phải `SELECT ... FOR UPDATE` trên dòng đơn hàng và validate transition hợp lệ (chỉ cho phép theo đúng thứ tự picked → shipped → delivered) trước khi ghi cả 3 boolean và enum, từ chối transition không hợp lệ với lỗi rõ ràng.
- Yêu cầu mọi write trong giai đoạn dual-write phải cùng 1 transaction ghi cả 3 cột boolean và cột `status`, không cho phép có thời điểm nào 2 cách biểu diễn lệch nhau dù đọc ở transaction khác đang chạy song song.
- Backfill dữ liệu lịch sử phải log rõ danh sách các đơn hàng có tổ hợp boolean không hợp lệ để team vận hành review tay, không tự động đoán và ghi sai enum.
- Sau khi chuyển toàn bộ luồng đọc/ghi sang cột `status`, quy định thời gian giữ lại 3 cột boolean cũ (dual-write một chiều để an toàn) trước khi được phép xóa hẳn, và điều kiện để rút ngắn thời gian này.
