# Deadlock detection & transaction isolation levels flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống web khác nhau (ngân hàng số, e-commerce, đặt phòng khách sạn, quản lý kho, mạng xã hội) để luyện việc phát hiện/xử lý deadlock giữa các transaction và chọn đúng isolation level cho từng tình huống đọc/ghi đồng thời.

---

## Chuyển tiền nội bộ giữa các tài khoản ngân hàng số

**Repository:** `deadlock-digital-bank-internal-transfer`

**Hệ thống:** Một app ngân hàng số cho phép user chuyển tiền giữa các tài khoản của chính họ và tài khoản người khác trong cùng hệ thống.

**Vai trò của flow:** Transaction cập nhật số dư của 2 tài khoản (trừ bên gửi, cộng bên nhận) phải lock đúng thứ tự để tránh deadlock khi nhiều lệnh chuyển tiền ngược hướng nhau chạy song song.

**Yêu cầu cụ thể:**
- Mô tả cụ thể kịch bản: transaction A chuyển từ account 1 → account 2, transaction B chuyển từ account 2 → account 1 chạy đồng thời; nếu mỗi transaction lock theo thứ tự "tài khoản nguồn trước, tài khoản đích sau" thì sẽ deadlock — yêu cầu code phải lock theo thứ tự cố định (ví dụ theo account_id tăng dần) để loại trừ deadlock này.
- Chọn isolation level `READ COMMITTED` hoặc cao hơn cho transaction cập nhật số dư, giải thích vì sao `REPEATABLE READ` mặc định của một số DB (MySQL/InnoDB) có thể gây phạm vi lock rộng hơn cần thiết và tăng khả năng deadlock trên các dòng không liên quan.
- Khi DB trả lỗi deadlock (ví dụ mã lỗi 40P01 ở Postgres, 1213 ở MySQL), transaction bị rollback phải được retry tự động tối đa 3 lần với backoff, không được để lỗi văng lên user như một giao dịch thất bại vĩnh viễn.
- Đảm bảo tổng số dư hai tài khoản trước và sau giao dịch không đổi (invariant) dù có retry hay không — viết test giả lập 50 giao dịch ngược hướng chạy song song và assert invariant này.
- Ghi log đầy đủ thông tin deadlock (transaction nào, lock gì, thời điểm) để phục vụ debug, không chỉ log "transaction failed".

---

## Đặt phòng khách sạn với cập nhật giá và tồn phòng cùng lúc

**Repository:** `deadlock-hotel-price-inventory-update`

**Hệ thống:** Một nền tảng đặt phòng khách sạn, admin khách sạn có thể cập nhật giá/số phòng trống trong khi khách đang đặt phòng.

**Vai trò của flow:** Transaction đặt phòng (trừ tồn phòng) và transaction admin cập nhật giá/tồn phòng cùng chạm vào bảng `room_inventory`, cần chọn isolation level và thứ tự lock hợp lý để không deadlock và không đọc dữ liệu giá sai (stale).

**Yêu cầu cụ thể:**
- Định nghĩa rõ transaction đặt phòng phải `SELECT ... FOR UPDATE` trên đúng các row (room_type, date) cần trừ tồn, không lock nguyên bảng.
- Nếu một request đặt phòng theo thứ tự ngày [10/9, 11/9, 12/9] và request khác đặt theo thứ tự [12/9, 11/9, 10/9] (đặt nhiều đêm cùng lúc), phải chuẩn hóa thứ tự lock theo khóa chính tăng dần trước khi lock để loại bỏ deadlock giữa 2 booking đa-ngày chồng lấn.
- So sánh và chọn isolation level: dùng `READ COMMITTED` + lock tường minh cho phần trừ tồn, tránh dùng `SERIALIZABLE` toàn bộ vì sẽ làm tăng abort rate không cần thiết ở phần đọc giá hiển thị cho khách đang xem trang.
- Khi admin sửa giá đúng lúc khách đang giữ lock trừ tồn cho phòng đó, request admin phải chờ hoặc bị timeout rõ ràng (không treo vô hạn) — quy định timeout cụ thể (ví dụ 3 giây) và hành vi khi timeout (báo lỗi "thử lại sau").
- Viết test dựng 2 transaction giả lập chồng lấn lock theo thứ tự ngược nhau, xác nhận hệ thống tự phát hiện/rollback một bên và request còn lại thành công.

---

## Cập nhật điểm số và thứ hạng leaderboard trong game/app học tập

**Repository:** `deadlock-game-leaderboard-update`

**Hệ thống:** Một app học tập gamification, nơi hoàn thành bài học cập nhật điểm user và một job nền tính lại thứ hạng leaderboard theo nhóm/lớp mỗi vài phút.

**Vai trò của flow:** Transaction cộng điểm khi user hoàn thành bài học và job tính lại rank (đọc/update nhiều user trong 1 nhóm) cạnh tranh lock trên bảng `user_score`, cần tránh deadlock giữa update đơn lẻ tần suất cao và job batch định kỳ.

**Yêu cầu cụ thể:**
- Yêu cầu job tính rank phải lock theo thứ tự user_id tăng dần trong từng nhóm, và xử lý theo từng nhóm nhỏ (không lock toàn bảng `user_score` một lần) để giảm collision với các transaction cộng điểm đang chạy real-time.
- Chỉ ra rủi ro cụ thể: nếu 2 user trong cùng nhóm cùng hoàn thành bài học đúng lúc job rank đang chạy, cả 2 transaction cộng điểm và transaction job đều có thể chờ nhau — yêu cầu retry tự động cho transaction cộng điểm (ưu tiên UX real-time của user) khi gặp deadlock, tối đa 2 lần trong 500ms.
- Chọn isolation level `READ COMMITTED` cho transaction cộng điểm để có UX nhanh, nhưng job tính rank cần đảm bảo tính nhất quán tương đối (không bắt buộc real-time chính xác tuyệt đối) — giải thích trade-off này bằng ví dụ cụ thể.
- Thiết kế cơ chế "lock timeout" ngắn (ví dụ 1 giây) cho transaction cộng điểm, để nếu job rank đang giữ lock lâu, user vẫn nhận được phản hồi (có thể là "điểm sẽ cập nhật sau ít giây") thay vì trải nghiệm bị treo.
- Đo và log tỷ lệ deadlock/lock-timeout theo giờ cao điểm (khi nhiều user học cùng lúc) để xác định xem có cần điều chỉnh tần suất chạy job rank hay không.

---

## Transaction checkout chạm nhiều bảng (đơn hàng, tồn kho, mã giảm giá) trên sàn e-commerce

**Repository:** `deadlock-ecommerce-order-inventory-coupon`

**Hệ thống:** Một sàn e-commerce mà mỗi lượt checkout tạo 1 đơn hàng, trừ tồn kho của nhiều sản phẩm trong giỏ, và áp dụng mã giảm giá dùng 1 lần (giới hạn số lượt sử dụng).

**Vai trò của flow:** Transaction checkout chạm 3 nhóm bảng khác nhau (order, inventory nhiều dòng sản phẩm, coupon) trong cùng 1 lượt, thứ tự chạm các bảng này lại phụ thuộc thứ tự sản phẩm trong giỏ hàng của từng khách nên rất dễ tạo deadlock chéo giữa các đơn hàng cạnh tranh cùng lúc.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: đơn hàng A có giỏ hàng gồm sản phẩm X rồi Y (khách thêm X trước), đơn hàng B có giỏ hàng gồm sản phẩm Y rồi X (khách thêm Y trước) — nếu code trừ tồn kho theo đúng thứ tự item xuất hiện trong giỏ của từng khách (không chuẩn hóa), A sẽ lock X trước rồi chờ lock Y, B sẽ lock Y trước rồi chờ lock X, tạo deadlock kinh điển; yêu cầu sort danh sách sản phẩm theo product_id tăng dần trước khi bắt đầu lock, bất kể thứ tự khách thêm vào giỏ.
- Chỉ ra rủi ro cross-table: nếu nhánh code validate mã giảm giá trước rồi mới trừ tồn kho ở 1 phần luồng (ví dụ đơn hàng dùng coupon), nhưng nhánh không dùng coupon lại trừ tồn kho trước rồi mới ghi nhận vào bảng order, thì 2 luồng có thể lock chéo giữa dòng coupon và các dòng inventory khi nhiều đơn hàng chạy song song — yêu cầu quy định 1 thứ tự lock cố định toàn cục cho mọi loại transaction checkout (ví dụ luôn: order row → inventory rows theo id tăng dần → coupon row cuối cùng), áp dụng nhất quán dù đơn có dùng coupon hay không.
- Mô tả cụ thể tình huống mùa sale: nhiều đơn hàng khác nhau chứa các sản phẩm trùng lặp một phần (đơn A có X, Y, Z; đơn B có Z, X) cùng chạy trong vài giây cao điểm — yêu cầu transaction bị deadlock phải tự động retry với backoff (tối đa 3 lần) và giới hạn thời gian tối đa cho 1 transaction checkout (ví dụ 2 giây) để tránh 1 đơn hàng bị treo kéo dài ảnh hưởng tới các đơn khác trong hàng đợi retry.
- Chọn isolation level `READ COMMITTED` kèm `SELECT ... FOR UPDATE` tường minh theo đúng thứ tự đã chuẩn hóa cho transaction checkout, giải thích vì sao dùng `SERIALIZABLE` cho toàn bộ transaction sẽ làm tăng abort rate không cần thiết trong giờ cao điểm khi nhiều đơn hàng không thực sự đụng chung sản phẩm nào.
- Viết test giả lập nhiều đơn hàng với thứ tự sản phẩm trong giỏ được hoán vị ngẫu nhiên (bao gồm cả trường hợp có/không dùng coupon), xác nhận không có transaction nào bị treo quá timeout quy định và invariant tồn kho + số lượt dùng coupon còn lại luôn đúng sau khi toàn bộ retry hoàn tất.

---

## Chuyển hàng liên kho hai chiều với nhiều SKU trong một lần chuyển

**Repository:** `deadlock-warehouse-bidirectional-transfer`

**Hệ thống:** Một hệ thống quản lý kho vận cho chuỗi cửa hàng/nhà phân phối, nhân viên tạo phiếu chuyển hàng giữa 2 kho (trừ tồn kho nguồn, cộng tồn kho đích), mỗi phiếu chuyển có thể gồm nhiều SKU khác nhau trong cùng 1 lần.

**Vai trò của flow:** Transaction xử lý phiếu chuyển hàng giống bài toán chuyển tiền giữa 2 tài khoản kinh điển, nhưng phức tạp hơn vì 1 giao dịch có thể động vào nhiều dòng tồn kho (nhiều cặp kho-SKU) cùng lúc, cần chọn đúng thứ tự lock để không deadlock khi có phiếu chuyển ngược chiều nhau chạy song song.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: phiếu P1 chuyển từ kho A sang kho B gồm SKU 101 rồi SKU 102 (theo thứ tự nhập trên phiếu); phiếu P2 chuyển từ kho B sang kho A gồm SKU 102 rồi SKU 101, cả 2 chạy song song — nếu mỗi transaction lock theo thứ tự "kho nguồn trước, rồi từng SKU theo thứ tự nhập trên phiếu", P1 sẽ lock (A,101) rồi chờ lock (B,102) trong khi P2 đã lock (B,102) rồi chờ lock (A,101), tạo deadlock; yêu cầu chuẩn hóa thứ tự lock toàn cục theo composite key (warehouse_id, sku_id) tăng dần, bất kể chiều chuyển hay thứ tự nhập trên phiếu.
- Với phiếu có nhiều SKU (ví dụ 20 SKU trong 1 lần chuyển), yêu cầu transaction phải lock toàn bộ các dòng tồn kho liên quan (cả nguồn lẫn đích) theo đúng thứ tự đã chuẩn hóa ngay từ đầu trước khi bắt đầu trừ/cộng bất kỳ dòng nào, không được lock rải rác từng SKU một trong lúc xử lý tuần tự (dễ tạo deadlock giữa các phiếu chỉ chồng lấn một phần SKU với nhau).
- Khi transaction bị rollback do deadlock, yêu cầu retry tự động nhưng phải đọc lại tồn kho hiện tại tại thời điểm retry (không dùng số liệu đã đọc ở lần thử trước), vì trong khoảng thời gian chờ retry có thể đã có phiếu chuyển khác của các kho liên quan làm thay đổi tồn kho.
- Chọn isolation level `READ COMMITTED` cho transaction chuyển kho, giải thích cụ thể vì sao `REPEATABLE READ` mặc định của một số DB có thể mở rộng phạm vi lock (gap lock/range lock) và ảnh hưởng tới các phiếu chuyển kho khác không hề liên quan tới cùng SKU, làm tăng khả năng deadlock giả tạo không cần thiết.
- Viết test dựng nhiều phiếu chuyển ngẫu nhiên giữa các cặp kho khác nhau với danh sách SKU chồng lấn một phần, chạy song song hàng loạt, assert tổng tồn kho từng SKU trên toàn hệ thống bất biến trước/sau và không có phiếu nào bị treo vượt quá timeout quy định dù đã tính cả thời gian retry.
