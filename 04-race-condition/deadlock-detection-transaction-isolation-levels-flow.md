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
