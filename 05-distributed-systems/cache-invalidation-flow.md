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

---

## Cache feed/trang cá nhân cho mạng xã hội

**Repository:** `cache-invalidation-social-feed-profile`

**Hệ thống:** Mạng xã hội cache feed trang chủ và trang cá nhân (profile) của user để giảm tải khi hiển thị bài viết, tránh phải chạy lại aggregation nặng mỗi lần load.

**Vai trò của flow:** Invalidate đúng cache khi user đăng bài mới, xóa bài, sửa bài, đổi thông tin profile — đặc biệt phải fan-out invalidate/refresh cho cache feed của tất cả follower đang xem nội dung của user đó.

**Yêu cầu cụ thể:**
- Khi user đăng bài mới, cache feed của chính họ (profile page) phải invalidate ngay, nhưng cache feed của hàng chục nghìn follower thì không thể invalidate đồng bộ tức thời — cần chiến lược phân biệt giữa fan-out-on-write (đẩy trước) cho user ít follower và fan-out-on-read (tính khi đọc) cho celebrity account có follower cực lớn, tránh nghẽn khi 1 bài đăng làm invalidate hàng triệu cache feed cùng lúc.
- Khi user xóa 1 bài viết, phải đảm bảo bài đó biến mất khỏi cache feed của follower trong thời gian hợp lý dù dùng fan-out-on-read hay pull — không được để bài đã xóa vẫn hiển thị dai dẳng do cache feed của follower ít khi active nên hiếm khi được refresh.
- Race condition: user vừa đăng bài mới vừa sửa bài cũ gần như đồng thời — invalidation event có thể tới cache theo thứ tự khác thứ tự thực tế (do khác hàng đợi/khác node xử lý), cần cơ chế đảm bảo thứ tự (version/timestamp) để cache feed không áp dụng nhầm invalidate cũ đè lên update mới.
- Khi user đổi avatar/tên hiển thị, các bài post cũ của họ đang nằm rải rác trong cache feed của follower khác vẫn hiển thị thông tin cũ (do cache feed thường lưu kèm snapshot info tác giả) — quyết định rõ có chấp nhận độ trễ hiển thị (denormalized data lệch tạm thời) hay bắt buộc phải invalidate toàn bộ cache liên quan, đánh đổi chi phí fan-out rất lớn.
- Đo lường: độ trễ trung bình và p99 từ lúc user đăng/xóa bài tới lúc feed của follower phản ánh đúng, và tỉ lệ follower vẫn thấy nội dung cũ quá X phút sau invalidation event, dùng để đánh giá hiệu quả chiến lược fan-out đang chọn.

---

## Cache nhiều tầng cho CMS/blog

**Repository:** `cache-invalidation-cms-multilayer`

**Hệ thống:** Hệ thống CMS/blog phục vụ bài viết qua nhiều tầng cache chồng lên nhau: CDN edge cache (gần user), cache tầng ứng dụng (application/object cache), và cache kết quả query DB (query cache) — mỗi tầng có TTL và cơ chế invalidate riêng.

**Vai trò của flow:** Đảm bảo khi 1 bài viết được sửa/xóa/publish, việc invalidate phải lan đúng và đủ qua toàn bộ các tầng cache, không chỉ dừng ở tầng gần nhất (application cache) mà bỏ sót CDN edge hoặc query cache khiến người đọc vẫn thấy nội dung cũ dù tầng app đã đúng.

**Yêu cầu cụ thể:**
- Khi biên tập viên sửa nội dung bài viết và publish lại, invalidation phải được gửi tới cả 3 tầng theo đúng thứ tự phụ thuộc (query cache trước, application cache sau, rồi purge CDN edge) — nếu purge CDN trước mà tầng dưới chưa cập nhật, request tiếp theo tới edge sẽ pull lại đúng nội dung cũ từ application cache và cache lại nó ở edge, coi như invalidate CDN vô nghĩa.
- Purge CDN edge cache thường không đồng bộ ngay lập tức trên toàn bộ điểm PoP (point of presence) toàn cầu — phải chấp nhận và công bố rõ "khoảng lan truyền" (propagation window) thực tế, đồng thời có cơ chế xác nhận purge đã tới đủ số PoP quan trọng trước khi coi invalidation hoàn tất.
- Với bài viết bị xóa hẳn (không chỉ sửa), phải đảm bảo tầng query cache (thường cache theo query như "danh sách bài mới nhất") cũng được invalidate, không chỉ cache theo ID bài viết đơn lẻ — nếu bỏ sót, bài đã xóa vẫn xuất hiện trong danh sách trang chủ dù trang chi tiết đã trả 404 đúng.
- Xử lý trường hợp một tầng cache invalidate thất bại (ví dụ API purge CDN timeout hoặc trả lỗi) — phải có retry với theo dõi trạng thái, và log rõ tầng nào invalidate thành công/thất bại để vận hành biết còn tầng nào đang phục vụ nội dung cũ, tránh tình trạng "tưởng đã invalidate hết" nhưng thực ra một tầng vẫn stale.
- Đo lường: với mỗi lần publish/sửa bài, track được "end-to-end staleness" — thời gian từ lúc publish tới lúc cả 3 tầng đều phản ánh đúng nội dung mới, phân theo từng tầng để biết tầng nào đang là nút thắt cổ chai của độ trễ invalidate.
