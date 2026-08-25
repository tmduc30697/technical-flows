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
