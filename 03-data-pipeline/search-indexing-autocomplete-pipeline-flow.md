# Search indexing/autocomplete pipeline flow — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — e-commerce, marketplace đa category, mạng xã hội, SaaS help center, nền tảng tuyển dụng, và bất động sản — nhằm luyện đủ các góc của flow lập chỉ mục tìm kiếm và gợi ý autocomplete (đồng bộ real-time, độ trễ index, xếp hạng kết quả, cá nhân hóa, chi phí vận hành index lớn).

---

## E-commerce tìm kiếm và gợi ý sản phẩm

**Repository:** `search-indexing-ecommerce-product-search`

**Hệ thống:** Một sàn e-commerce với hàng triệu sản phẩm, cần thanh tìm kiếm với autocomplete và kết quả tìm kiếm chính xác theo tên/thuộc tính sản phẩm.

**Vai trò của flow:** Đồng bộ dữ liệu sản phẩm (tên, giá, tồn kho, danh mục) từ database chính vào search index, và phục vụ autocomplete/tìm kiếm với độ trễ thấp.

**Yêu cầu cụ thể:**
- Khi sản phẩm thay đổi giá hoặc hết hàng, search index phải được cập nhật trong vòng vài giây, tránh hiển thị giá cũ hoặc cho phép tìm thấy sản phẩm đã hết hàng mà không có cảnh báo rõ ràng.
- Autocomplete phải trả kết quả trong thời gian rất ngắn (dưới 100ms) và ưu tiên gợi ý sản phẩm đang bán tốt/còn hàng lên trước, không chỉ khớp chuỗi ký tự đơn thuần.
- Index phải xử lý được lỗi chính tả và từ đồng nghĩa phổ biến (ví dụ "tivi"/"tv", dấu tiếng Việt có/không dấu) để không bỏ lỡ kết quả đúng khi người dùng gõ sai/thiếu dấu.
- Khi có cập nhật hàng loạt (bulk update giá dịp sale lớn ảnh hưởng hàng trăm nghìn sản phẩm), pipeline đồng bộ vào index không được làm chậm hoặc gây downtime cho tìm kiếm đang phục vụ người dùng thật.
- Có cơ chế phát hiện và tự phục hồi khi index bị lệch so với database nguồn (ví dụ do lỗi đồng bộ), thông qua job đối soát định kỳ chạy full reindex một phần hoặc toàn phần mà không ảnh hưởng tìm kiếm đang chạy (dùng index song song rồi swap).

---

## Mạng xã hội tìm kiếm người dùng, hashtag và bài viết

**Repository:** `search-indexing-social-user-hashtag-post`

**Hệ thống:** Một mạng xã hội cần chức năng tìm kiếm người dùng, hashtag, và nội dung bài viết, kết hợp gợi ý autocomplete cá nhân hóa và trending.

**Vai trò của flow:** Lập chỉ mục liên tục nội dung mới (bài viết, hashtag) và hồ sơ người dùng, phục vụ tìm kiếm kết hợp mức độ liên quan, độ phổ biến, và cá nhân hóa theo người tìm.

**Yêu cầu cụ thể:**
- Bài viết mới đăng phải xuất hiện được trong kết quả tìm kiếm trong thời gian rất ngắn (gần thời gian thực) để phục vụ các sự kiện đang nóng (breaking news, viral moment), khác với yêu cầu index chậm hơn có thể chấp nhận ở catalog sản phẩm tĩnh.
- Autocomplete tìm người dùng phải cá nhân hóa theo mối quan hệ của người tìm (ưu tiên bạn bè/người đang follow lên trước người lạ cùng tên), không chỉ xếp hạng theo độ phổ biến chung.
- Hashtag trending phải được tính toán và cập nhật gần thời gian thực dựa trên tốc độ tăng trưởng sử dụng, không phải chỉ dựa trên tổng số lượt dùng lịch sử (tránh hashtag cũ dùng nhiều nhưng đã hết hot vẫn đứng đầu).
- Bài viết/tài khoản bị xóa hoặc bị khóa do vi phạm chính sách phải biến mất khỏi kết quả tìm kiếm và autocomplete ngay lập tức, đồng bộ với hành động kiểm duyệt nội dung, không có khoảng trễ để nội dung vi phạm vẫn tìm được.
- Index phải chịu được khối lượng ghi rất lớn liên tục (hàng nghìn bài viết/giây ở giờ cao điểm) mà không làm chậm độ trễ đọc (tìm kiếm) của người dùng khác đang truy vấn cùng lúc.

---

## Nền tảng bất động sản tìm kiếm theo bản đồ và autocomplete địa chỉ

**Repository:** `search-indexing-real-estate-map-autocomplete`

**Hệ thống:** Một nền tảng đăng tin bất động sản cho phép người tìm mua/thuê tra cứu theo khu vực trên bản đồ kết hợp autocomplete gợi ý địa chỉ/khu vực.

**Vai trò của flow:** Lập chỉ mục tin bất động sản kèm dữ liệu địa lý (tọa độ, ranh giới khu vực), phục vụ tìm kiếm theo vùng bản đồ và autocomplete địa chỉ chính xác.

**Yêu cầu cụ thể:**
- Tìm kiếm theo vùng hiển thị trên bản đồ (bounding box/bán kính từ một điểm) phải trả đúng và đủ các tin nằm trong vùng đó, xử lý chính xác các trường hợp tin ở biên giới vùng (gần đúng ranh giới quận/khu vực).
- Autocomplete địa chỉ phải xử lý được cách viết địa chỉ không thống nhất (viết tắt, thiếu dấu, thứ tự phường/quận/thành phố khác nhau) và gợi ý đúng địa chỉ chuẩn hóa, không yêu cầu người dùng gõ chính xác định dạng địa chỉ đầy đủ.
- Khi người đăng tin cập nhật tọa độ/vị trí của bất động sản (ví dụ sửa lại vị trí đánh dấu trên bản đồ cho chính xác hơn), index phải cập nhật ngay để tin xuất hiện đúng ở kết quả tìm kiếm theo vùng bản đồ mới, biến mất khỏi vùng cũ.
- Tin đã cho thuê/bán xong phải được gỡ khỏi kết quả tìm kiếm theo bản đồ ngay lập tức, tránh người tìm liên hệ vào tin không còn hiệu lực gây trải nghiệm xấu.
- Khi người dùng zoom/pan bản đồ liên tục để khám sát khu vực, hệ thống phải trả kết quả tìm kiếm theo vùng mới với độ trễ thấp cho từng lần thay đổi vùng nhìn, không tính toán lại toàn bộ từ đầu mỗi lần một cách tốn kém tài nguyên.
