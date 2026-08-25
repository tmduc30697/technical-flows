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

## Marketplace tìm kiếm đa dạng category với faceted search

**Repository:** `search-indexing-marketplace-faceted-search`

**Hệ thống:** Một marketplace bán đa dạng ngành hàng (điện tử, thời trang, nội thất...) với thuộc tính lọc (facet) khác nhau tùy category.

**Vai trò của flow:** Lập chỉ mục sản phẩm cùng với các thuộc tính lọc động theo từng category, phục vụ tìm kiếm kết hợp bộ lọc (giá, thương hiệu, kích thước...) trả kết quả nhanh và đúng.

**Yêu cầu cụ thể:**
- Schema index phải hỗ trợ thuộc tính động khác nhau theo từng category (ví dụ "dung lượng RAM" cho điện tử, "size" cho thời trang) mà không cần định nghĩa cứng trước toàn bộ trường cho mọi loại sản phẩm.
- Kết hợp nhiều bộ lọc cùng lúc (giá + thương hiệu + đánh giá) phải trả kết quả đúng và số lượng sản phẩm khớp với từng lựa chọn filter hiển thị (facet count) phải chính xác theo thời gian thực, không hiển thị số liệu cũ gây hiểu lầm cho người mua.
- Khi seller thêm/sửa sản phẩm với thuộc tính mới chưa từng xuất hiện trong category đó, pipeline phải cập nhật được mapping index động mà không cần downtime để rebuild toàn bộ index.
- Sản phẩm bị seller ẩn hoặc bị gỡ do vi phạm phải biến mất khỏi kết quả tìm kiếm ngay lập tức (độ trễ tối thiểu), không thể chấp nhận độ trễ index lớn cho trường hợp gỡ bỏ nội dung vi phạm.
- Chi phí vận hành index (dung lượng, tài nguyên tính toán facet) phải được kiểm soát khi số lượng sản phẩm và thuộc tính tăng theo thời gian — cần chiến lược archive/loại bỏ sản phẩm không còn hoạt động khỏi index chính để tối ưu chi phí.

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

## SaaS help center tìm kiếm tài liệu hỗ trợ đa ngôn ngữ

**Repository:** `search-indexing-saas-help-center-multilingual`

**Hệ thống:** Một SaaS B2B cung cấp trung tâm trợ giúp (help center/knowledge base) với bài viết hướng dẫn được viết ở nhiều ngôn ngữ cho khách hàng toàn cầu.

**Vai trò của flow:** Lập chỉ mục nội dung tài liệu hỗ trợ theo từng ngôn ngữ, phục vụ tìm kiếm ngữ nghĩa (không chỉ khớp từ khóa chính xác) giúp khách hàng tìm đúng bài viết cần.

**Yêu cầu cụ thể:**
- Tìm kiếm phải trả kết quả theo đúng ngôn ngữ người dùng đang sử dụng (hoặc ngôn ngữ họ chọn), không trộn lẫn kết quả từ bài viết ngôn ngữ khác dù nội dung có thể liên quan.
- Phải xử lý được trường hợp một bài viết được cập nhật ở một ngôn ngữ nhưng bản dịch ở ngôn ngữ khác chưa cập nhật theo — index cần phản ánh đúng phiên bản/thời điểm cập nhật của từng ngôn ngữ riêng biệt.
- Tìm kiếm phải hiểu được ý định câu hỏi thay vì chỉ khớp từ khóa chính xác (ví dụ người dùng gõ câu hỏi tự nhiên "làm sao để hủy đăng ký" phải tìm ra bài viết có nội dung liên quan dù không chứa chính xác các từ đó).
- Khi một bài viết được đánh dấu lỗi thời/deprecated bởi đội content, nó phải bị hạ hạng hoặc loại khỏi kết quả tìm kiếm ngay, tránh khách hàng làm theo hướng dẫn không còn đúng.
- Theo dõi được các truy vấn tìm kiếm không trả ra kết quả phù hợp (zero-result hoặc bị người dùng bỏ qua hết các kết quả) để đội content biết cần viết thêm bài viết nào, biến pipeline tìm kiếm thành nguồn dữ liệu cải thiện nội dung liên tục.

---

## Nền tảng tuyển dụng tìm kiếm việc làm với bộ lọc phức tạp

**Repository:** `search-indexing-job-search-complex-filters`

**Hệ thống:** Một nền tảng tuyển dụng cho phép ứng viên tìm việc theo nhiều tiêu chí (mức lương, địa điểm, kỹ năng, loại hợp đồng) trên hàng trăm nghìn tin tuyển dụng.

**Vai trò của flow:** Lập chỉ mục tin tuyển dụng kèm các trường lọc có cấu trúc, phục vụ tìm kiếm kết hợp filter phức tạp và xếp hạng theo độ liên quan lẫn độ mới của tin.

**Yêu cầu cụ thể:**
- Tin tuyển dụng hết hạn hoặc đã đủ số lượng tuyển phải tự động biến mất khỏi kết quả tìm kiếm đúng thời điểm hết hạn, không phụ thuộc vào việc nhà tuyển dụng có chủ động gỡ tin hay không.
- Xếp hạng kết quả phải cân bằng giữa độ liên quan với từ khóa tìm và độ mới của tin (tin mới đăng cần có cơ hội hiển thị dù chưa có nhiều lượt xem/ứng tuyển để làm tín hiệu xếp hạng), tránh tình trạng chỉ các tin cũ có nhiều tương tác luôn đứng đầu.
- Bộ lọc mức lương phải xử lý đúng các tin ghi lương theo nhiều đơn vị khác nhau (theo giờ, theo tháng, theo năm, hoặc "thỏa thuận") khi so khớp với khoảng lương ứng viên chọn lọc, không bỏ sót hoặc hiển thị sai tin do đơn vị không khớp.
- Khi nhà tuyển dụng sửa tin đã đăng (đổi mức lương, đổi địa điểm), index phải cập nhật ngay để ứng viên đang lọc theo tiêu chí cũ không thấy tin đã không còn phù hợp với bộ lọc họ chọn.
- Lưu lại và tận dụng được lịch sử tìm kiếm/lọc của ứng viên để cải thiện gợi ý autocomplete cá nhân hóa (ví dụ ưu tiên gợi ý địa điểm/ngành nghề họ tìm thường xuyên), có cơ chế cho ứng viên xóa lịch sử này nếu muốn.

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
