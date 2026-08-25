# News feed/timeline fan-out (write vs read) flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần phân phối nội dung tới nhiều người đọc — feed mạng xã hội với vấn đề celebrity, notification feed SaaS, feed deal cá nhân hóa e-commerce, và fan-out tin nhắn group chat lớn — nhằm luyện cách chọn giữa fan-out-on-write và fan-out-on-read, xử lý ngưỡng chuyển chiến lược, và giữ đúng thứ tự hiển thị trong web app thực tế.

---

## Feed mạng xã hội kiểu Twitter/X với vấn đề celebrity

**Repository:** `feed-fanout-social-celebrity-problem`

**Hệ thống:** Mạng xã hội nơi user follow nhau, mỗi khi có bài viết mới cần xuất hiện trong feed của người follow.

**Vai trò của flow:** Fan-out-on-write (đẩy bài viết vào feed của từng follower ngay khi đăng) kết hợp fan-out-on-read cho tài khoản celebrity (follower quá lớn) để tránh viết hàng triệu feed cùng lúc.

**Yêu cầu cụ thể:**
- User thường (follower ít) dùng fan-out-on-write: bài viết mới được đẩy (push) ngay vào feed cache của từng follower để đọc feed cực nhanh (chỉ đọc 1 danh sách đã build sẵn).
- Tài khoản celebrity (ví dụ trên 1 triệu follower) phải chuyển sang fan-out-on-read: không ghi vào feed của từng follower (quá tốn tài nguyên), mà khi follower load feed, hệ thống merge real-time bài viết của các celebrity họ follow vào feed.
- Định nghĩa rõ ngưỡng số follower để một tài khoản được coi là "celebrity" và tự động chuyển chiến lược, và xử lý mượt khi tài khoản vừa vượt ngưỡng (không mất bài viết cũ đã fan-out theo cách cũ).
- Độ trễ hiển thị bài viết mới trong feed follower phải đo được và có SLA riêng cho từng chiến lược (fan-out-on-write nên gần real-time; fan-out-on-read có thể chấp nhận build feed chậm hơn một chút khi load).
- Xử lý được trường hợp user follow rất nhiều celebrity (fan-out-on-read phải merge nhiều nguồn hiệu quả, không quét tuần tự từng celebrity một cách chậm).

---

## Feed thông báo (notification feed) cho platform SaaS

**Repository:** `feed-fanout-saas-notification-feed`

**Hệ thống:** Một SaaS gửi thông báo trong app (in-app notification) cho user khi có sự kiện liên quan tới họ (được mention, được assign task).

**Vai trò của flow:** Fan-out sự kiện tới đúng danh sách user cần nhận thông báo, quyết định push ngay (write) hay tính toán lúc user mở trang thông báo (read) tùy loại sự kiện và số người liên quan.

**Yêu cầu cụ thể:**
- Sự kiện liên quan tới ít người (ví dụ được mention trong comment, chỉ vài người xem) nên fan-out-on-write ngay để hiển thị tức thời, không cần tính toán lại mỗi lần user mở trang.
- Sự kiện liên quan tới rất nhiều người (ví dụ thông báo toàn công ty/broadcast) nên tránh ghi riêng cho từng user (tốn storage/thời gian) mà lưu 1 bản chung và tính "đã đọc/chưa đọc" per-user riêng biệt.
- Đảm bảo thứ tự thông báo hiển thị cho user đúng theo thời gian xảy ra sự kiện, dù được ghi vào feed theo 2 chiến lược khác nhau (write-fan-out và broadcast chung) — phải merge đúng thứ tự khi hiển thị.
- Trạng thái "đã đọc" phải được cập nhật chính xác và nhất quán dù user đọc trên nhiều thiết bị (web, mobile) gần như đồng thời, không bị hiển thị lại "chưa đọc" sau khi đã đọc ở thiết bị khác.
- Đo lường: thời gian từ lúc sự kiện xảy ra tới lúc xuất hiện trong notification feed của user (fan-out latency), theo từng loại sự kiện.

---

## Feed deal/gợi ý cá nhân hóa cho e-commerce

**Repository:** `feed-fanout-ecommerce-personalized-deals`

**Hệ thống:** Trang e-commerce hiển thị feed sản phẩm/deal được cá nhân hóa cho từng user dựa trên hành vi mua sắm, cập nhật liên tục theo hoạt động mới (flash sale mới, sản phẩm mới theo sở thích).

**Vai trò của flow:** Quyết định chiến lược fan-out cho feed cá nhân hóa — vì nội dung phụ thuộc rất nhiều vào từng user (không đơn giản là follow ai), nên chủ yếu tính toán lúc đọc (fan-out-on-read) với cache kết quả gần nhất.

**Yêu cầu cụ thể:**
- Feed phải được tính toán (hoặc lấy từ cache đã tính trước - precompute) khi user mở app, không phải fan-out-on-write cho từng deal mới tới tất cả user vì chi phí cá nhân hóa cho mỗi user là riêng biệt và tốn kém.
- Có cơ chế precompute/cache feed cho user active thường xuyên (tính trước và refresh định kỳ) để giảm độ trễ khi họ mở app, trong khi user ít hoạt động có thể tính on-demand khi họ vào.
- Khi có deal/flash sale khẩn cấp cần hiển thị ngay cho một nhóm user cụ thể (ví dụ user đã từng xem sản phẩm đó), phải có đường tắt inject vào feed cache đã tính trước mà không cần đợi chu kỳ tính toán lại toàn bộ.
- Đảm bảo feed không hiển thị sản phẩm đã hết hàng/deal đã kết thúc dù đã được tính toán trước (cần một bước lọc "tính hợp lệ tại thời điểm hiển thị" trước khi trả về user).
- Đo lường: độ trễ tính từ lúc user mở feed tới lúc nhận được kết quả (đặc biệt với user chưa có cache sẵn — cold start), và tối ưu để không vượt ngưỡng UX chấp nhận được (ví dụ dưới 500ms).

---

## Fan-out tin nhắn trong group chat lớn

**Repository:** `feed-fanout-large-group-chat`

**Hệ thống:** App chat có group chat với số lượng thành viên lớn (hàng nghìn người, kiểu channel/community), mỗi tin nhắn cần đến được mọi thành viên đang online và lưu lại cho người offline đọc sau.

**Vai trò của flow:** Fan-out tin nhắn tới thành viên online theo thời gian thực (qua kết nối realtime/websocket), đồng thời lưu vào lịch sử chung để fan-out-on-read cho thành viên offline khi họ mở lại app.

**Yêu cầu cụ thể:**
- Với group nhỏ, có thể push tin nhắn trực tiếp tới connection của từng thành viên online; với group cực lớn (hàng nghìn người online cùng lúc), cần chiến lược phân tầng (ví dụ broadcast qua pub/sub theo channel, không loop gửi từng người) để không nghẽn server.
- Thành viên offline không cần fan-out ngay — khi họ mở lại app, hệ thống đọc trực tiếp lịch sử tin nhắn chung của group (fan-out-on-read) từ vị trí họ đã đọc tới lần cuối (last-read-position).
- Đảm bảo tin nhắn hiển thị đúng thứ tự cho mọi người dù được gửi tới bằng 2 cơ chế khác nhau (real-time push cho người online, đọc lịch sử cho người offline) — không có sai lệch thứ tự giữa 2 luồng.
- Xử lý được trường hợp thành viên rời/vào group liên tục (group động) — người mới vào chỉ thấy tin nhắn từ lúc họ vào (hoặc theo policy được cấu hình), không fan-out lịch sử cũ không liên quan.
- Đo lường: độ trễ gửi tin nhắn tới thành viên online (mục tiêu dưới 200-300ms) và khả năng chịu tải khi có group đông thành viên cùng hoạt động (ví dụ 5000 thành viên online cùng chat sôi động).
