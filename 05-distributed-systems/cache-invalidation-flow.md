# Cache invalidation flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần giữ cache đồng bộ với dữ liệu gốc — trang chi tiết sản phẩm e-commerce, profile/feed mạng xã hội, CMS/blog nhiều tầng cache, permission cache cho phân quyền, và giá/tồn kho flash sale — nhằm luyện chiến lược invalidate theo event, xử lý race condition, và tránh cache stampede trong web app thực tế.

---

## Cache trang chi tiết sản phẩm cho e-commerce

**Repository:** `cache-invalidation-ecommerce-product-page`

**Hệ thống:** Trang e-commerce cache HTML/dữ liệu trang chi tiết sản phẩm (giá, tồn kho, mô tả) để giảm tải DB cho lượng truy cập lớn.

**Vai trò của flow:** Invalidate đúng cache khi sản phẩm được cập nhật (đổi giá, hết hàng) để không hiển thị thông tin sai cho khách, đồng thời giữ hiệu quả cache cao khi sản phẩm không đổi.

**Yêu cầu cụ thể:**
- Khi giá hoặc tồn kho sản phẩm thay đổi ở hệ thống quản trị, cache liên quan phải được invalidate trong vòng vài giây, không dựa hoàn toàn vào TTL cố định dài (ví dụ TTL 1 giờ là không đủ nhanh cho thay đổi giá).
- Có chiến lược rõ giữa cache invalidation theo event (khi có thay đổi, chủ động xóa) và TTL ngắn dự phòng (để tránh trường hợp event bị lỡ vẫn tự hết hạn sau một khoảng thời gian giới hạn).
- Xử lý race condition: request đọc cache đúng lúc dữ liệu đang được invalidate/update không được thấy trạng thái nửa cũ nửa mới (ví dụ giá mới nhưng tồn kho cũ).
- Với sản phẩm có traffic rất cao (best-seller), invalidate không được gây "cache stampede" (mọi request cùng lúc miss cache dội xuống DB) — cần cơ chế warm lại cache có kiểm soát (single flight hoặc pre-warm).
- Đo lường: cache hit ratio theo loại trang, và độ trễ giữa lúc dữ liệu gốc đổi tới lúc cache phản ánh đúng (staleness window thực tế).

---

## Cache profile/feed cho mạng xã hội

**Repository:** `cache-invalidation-social-profile-feed`

**Hệ thống:** Mạng xã hội cache thông tin profile và feed bài viết để giảm tải khi user xem trang cá nhân/newsfeed liên tục.

**Vai trò của flow:** Invalidate cache khi user cập nhật profile hoặc có bài viết mới, cân bằng giữa độ mới của dữ liệu và hiệu năng đọc ở quy mô rất lớn.

**Yêu cầu cụ thể:**
- Khi user đổi avatar/tên hiển thị, cache profile của chính họ phải invalidate ngay, nhưng cache trong feed của người khác (nơi hiển thị tên/avatar cũ của họ trong bài post cũ) có thể chấp nhận độ trễ ngắn (vài phút) — nêu rõ mức độ trễ chấp nhận được cho từng loại cache.
- Với user có lượng follower rất lớn (celebrity), việc invalidate cache profile không được gây spike tải đột ngột lên DB do hàng loạt cache ở nhiều nơi cùng miss đồng thời.
- Định nghĩa rõ cấp độ cache nào bắt buộc invalidate ngay (dữ liệu bảo mật/quyền riêng tư, ví dụ đổi trạng thái tài khoản private) và cấp độ nào có thể chấp nhận eventually consistent (số lượng like, follower count).
- Xử lý cache ở tầng CDN (cache HTML/API response ở edge) khác với cache ở tầng application (Redis) — phải có chiến lược invalidate riêng cho từng tầng vì CDN thường có độ trễ purge cao hơn.
- Có công cụ debug cho phép kiểm tra nhanh "vì sao dữ liệu này chưa cập nhật" — trace được cache đang nằm ở tầng nào, TTL còn lại bao nhiêu.

---

## Cache trang CMS/blog với nhiều tầng cache (CDN + application)

**Repository:** `cache-invalidation-cms-multi-layer`

**Hệ thống:** Trang tin tức/blog cache nội dung bài viết ở nhiều tầng (CDN edge, application cache, browser cache) để phục vụ lượng đọc lớn với chi phí thấp.

**Vai trò của flow:** Invalidate đồng bộ giữa các tầng cache khi bài viết được sửa/xóa/gỡ (ví dụ do lỗi biên tập cần gỡ khẩn), đảm bảo không còn tầng nào phục vụ nội dung cũ/đã gỡ.

**Yêu cầu cụ thể:**
- Khi biên tập viên bấm "gỡ bài" (unpublish), request purge phải được gửi tới tất cả tầng cache (CDN, app cache) và xác nhận thành công ở toàn bộ, không chỉ "bắn và quên" (fire-and-forget) vì bài gỡ khẩn thường liên quan tới vấn đề pháp lý/nhạy cảm.
- Có timeout/retry rõ ràng cho việc purge CDN (một số CDN provider purge không tức thời) và có cách kiểm tra xác nhận đã purge thành công thực sự (không chỉ tin API trả 200).
- Với sửa nội dung thông thường (không khẩn cấp), có thể chấp nhận cache tồn tại thêm vài phút để tối ưu chi phí purge, nhưng phải phân biệt rõ 2 loại action (gỡ khẩn vs sửa thường) trong hệ thống.
- Xử lý được trường hợp một bài viết có nhiều URL/biến thể cache (AMP version, mobile version, các ngôn ngữ khác nhau) — invalidate phải cover đủ tất cả biến thể liên quan tới bài viết đó.
- Có audit log ghi lại mọi lần purge cache khẩn cấp (ai bấm, bài nào, lúc nào) để phục vụ điều tra khi có sự cố nội dung.

---

## Cache session/permission cho hệ thống phân quyền

**Repository:** `cache-invalidation-session-permission`

**Hệ thống:** Một SaaS B2B cache thông tin quyền hạn (permission/role) của user để tránh query DB mỗi request, ảnh hưởng tới việc kiểm soát truy cập.

**Vai trò của flow:** Invalidate cache permission ngay khi quyền của user bị thay đổi (bị revoke quyền, đổi role), vì đây là dữ liệu bảo mật — cache cũ có thể dẫn tới lỗ hổng cho phép truy cập trái phép.

**Yêu cầu cụ thể:**
- Khi admin revoke quyền của một user, cache permission liên quan phải bị invalidate ngay lập tức và có xác nhận (không chấp nhận eventual consistency dạng "vài giây sau mới có hiệu lực" cho việc revoke quyền).
- Trong kiến trúc nhiều instance service cùng cache permission cục bộ, phải có cơ chế broadcast invalidation tới toàn bộ instance (ví dụ qua pub/sub) khi có thay đổi quyền, không chỉ invalidate ở một instance xử lý request đổi quyền.
- Nếu broadcast invalidation thất bại một phần (một instance không nhận được message do mất kết nối tạm thời), phải có cơ chế dự phòng (TTL rất ngắn cho cache permission, ví dụ vài giây) để giới hạn "cửa sổ rủi ro" tối đa.
- Có test case cụ thể: user đang có session hoạt động bị revoke quyền admin ngay khi đang dùng — request tiếp theo của họ (sau invalidation) phải bị chặn đúng theo quyền mới, không dùng permission cache cũ.
- Log đầy đủ mọi lần invalidate permission cache (ai, quyền gì, khi nào) để phục vụ audit an ninh khi cần điều tra truy cập trái phép.

---

## Cache giá/tồn kho trong flash sale với tần suất thay đổi cực cao

**Repository:** `cache-invalidation-flash-sale-price-inventory`

**Hệ thống:** E-commerce trong giờ flash sale có giá và tồn kho thay đổi liên tục (giảm giá theo giờ, tồn kho giảm nhanh), cache phải theo kịp thay đổi mà vẫn chịu được tải đọc khổng lồ.

**Vai trò của flow:** Invalidate cache tồn kho/giá gần như real-time trong khi vẫn giữ được throughput đọc cao, khác với cache thông thường có thể chấp nhận độ trễ lớn hơn.

**Yêu cầu cụ thể:**
- Cache tồn kho phải có TTL rất ngắn (giây) kết hợp active invalidation khi có đơn hàng trừ kho, để giảm thiểu rủi ro hiển thị "còn hàng" sai trong lúc sản phẩm sắp hết.
- Với sản phẩm sắp hết hàng (dưới ngưỡng an toàn), phải chuyển sang đọc trực tiếp từ nguồn (bỏ qua cache) để đảm bảo chính xác, đánh đổi tăng tải DB cho nhóm sản phẩm nhỏ này để bù lại độ chính xác.
- Xử lý cache stampede khi giá flash sale bắt đầu/kết thúc đúng giờ (hàng loạt cache cùng invalidate đồng thời tại thời điểm giờ chốt) — cần pre-warm cache trước thời điểm đổi giá vài giây để tránh dội tải.
- Đảm bảo giá hiển thị cho user tại thời điểm checkout khớp với giá thực tế lúc submit đơn (double-check giá ở backend, không tin hoàn toàn giá đã cache hiển thị ở client).
- Đo lường: tỉ lệ hiển thị sai giá/tồn kho (nếu có) trong log thực tế của các lần flash sale trước, dùng làm baseline để cải thiện chiến lược invalidation.
