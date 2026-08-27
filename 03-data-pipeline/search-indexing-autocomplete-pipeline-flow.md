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

---

## Marketplace đa category với facet lọc khác biệt hoàn toàn

**Repository:** `search-indexing-marketplace-multi-category-facet`

**Hệ thống:** Một marketplace bán hàng đa category hoàn toàn khác biệt (điện tử, thời trang, nội thất...), mỗi category có bộ thuộc tính/facet lọc riêng.

**Vai trò của flow:** Lập chỉ mục sản phẩm từ nhiều category với schema thuộc tính khác nhau, phục vụ tìm kiếm xuyên category và autocomplete gợi ý đúng theo ngữ cảnh category đang tìm.

**Yêu cầu cụ thể:**
- Mỗi category có bộ facet lọc riêng (điện tử có "dung lượng RAM", thời trang có "size/màu", nội thất có "chất liệu") — index phải lưu và truy vấn hiệu quả với schema thuộc tính động theo category mà không phải tạo index riêng biệt cho từng category gây khó khăn khi tìm xuyên category.
- Khi người dùng gõ một từ khóa mơ hồ có thể thuộc nhiều category (ví dụ "bàn" có thể là bàn ăn nội thất hoặc liên quan đến bàn phím máy tính), autocomplete phải suy luận được ngữ cảnh category phù hợp dựa trên hành vi tìm kiếm/lịch sử, tránh gợi ý lẫn lộn không liên quan gây khó chịu.
- Khi sản phẩm được người bán đổi category sau khi đã đăng, index phải cập nhật đồng thời cả danh mục lẫn bộ facet thuộc tính tương ứng, đảm bảo sản phẩm không còn xuất hiện ở bộ lọc facet của category cũ không còn phù hợp.
- Kết quả tìm kiếm xuyên category (ví dụ tìm "quà tặng sinh nhật" trả về cả điện tử, thời trang, đồ chơi) phải xếp hạng công bằng giữa các category khác nhau về bản chất — độ khớp từ khóa của một sản phẩm thời trang không thể so trực tiếp với một sản phẩm điện tử theo cùng công thức điểm số mà không tính đến đặc thù từng category.
- Khi thêm một category hoàn toàn mới vào hệ thống, việc bổ sung schema/facet mới cho category đó không được yêu cầu reindex toàn bộ dữ liệu các category hiện có đang phục vụ tìm kiếm sống.

---

## SaaS help center tìm kiếm ngữ nghĩa tài liệu trợ giúp

**Repository:** `search-indexing-saas-help-center-semantic`

**Hệ thống:** Một SaaS có help center/tài liệu kỹ thuật cho khách hàng tự tra cứu, nội dung được đội support/docs chỉnh sửa thường xuyên.

**Vai trò của flow:** Lập chỉ mục nội dung bài viết trợ giúp, phục vụ tìm kiếm xếp hạng theo mức độ liên quan ngữ nghĩa (không chỉ khớp từ khóa) và đảm bảo cập nhật ngay khi nội dung được sửa để tránh hiển thị thông tin lỗi thời.

**Yêu cầu cụ thể:**
- Kết quả tìm kiếm phải xếp hạng được các bài viết diễn đạt khác từ ngữ nhưng cùng ý nghĩa với câu hỏi người dùng gõ (ví dụ hỏi "không đăng nhập được" phải tìm ra bài viết tiêu đề "khắc phục lỗi xác thực tài khoản"), đòi hỏi cơ chế đánh giá liên quan vượt ra ngoài khớp từ khóa thuần túy, đồng thời không được làm chậm đáng kể thời gian trả kết quả so với tìm kiếm từ khóa thông thường.
- Khi một bài viết được đội docs chỉnh sửa để sửa thông tin sai/lỗi thời, index phải phản ánh nội dung mới gần như ngay lập tức — độ trễ cập nhật kéo dài có thể khiến khách hàng tra cứu ra hướng dẫn đã biết là sai, gây thêm ticket support thay vì giảm.
- Bài viết ở trạng thái nháp/chưa xuất bản hoặc đã bị gỡ do lỗi nghiêm trọng phải không xuất hiện trong kết quả tìm kiếm, kể cả trong khoảng thời gian ngắn giữa lúc tác giả bấm lưu nháp và lúc job index xử lý xong, tránh rò rỉ nội dung chưa hoàn chỉnh/sai ra khách hàng.
- Khi nhiều phiên bản của cùng một bài viết tồn tại cho các gói/phiên bản sản phẩm khác nhau, kết quả tìm kiếm phải trả đúng phiên bản phù hợp với ngữ cảnh khách hàng đang xem, tránh gợi ý bài hướng dẫn cho tính năng khách hàng không có quyền sử dụng.
- Việc đội docs sửa hàng loạt bài viết cùng lúc (ví dụ đổi tên tính năng, rebrand) phải được xử lý qua pipeline index mà không tạo ra khoảng thời gian index ở trạng thái nửa cũ nửa mới hiển thị lẫn lộn cho các khách hàng khác nhau tìm kiếm cùng lúc.

---

## Nền tảng tuyển dụng với yêu cầu độ tươi mới của tin đăng

**Repository:** `search-indexing-job-board-listing-freshness`

**Hệ thống:** Một nền tảng tuyển dụng cho phép ứng viên tìm việc theo kỹ năng/địa điểm/mức lương, nhà tuyển dụng đăng tin và có thể đóng tin bất cứ lúc nào khi đã tuyển đủ.

**Vai trò của flow:** Lập chỉ mục tin tuyển dụng với các thuộc tính lọc (kỹ năng, địa điểm, mức lương), phục vụ tìm kiếm và đảm bảo tin hết hạn/đã tuyển đủ biến mất khỏi kết quả ngay lập tức.

**Yêu cầu cụ thể:**
- Khi nhà tuyển dụng đánh dấu tin đã tuyển đủ hoặc chủ động đóng tin, tin đó phải biến mất khỏi kết quả tìm kiếm ngay lập tức, không chờ tới chu kỳ đồng bộ định kỳ — vì ứng viên nộp hồ sơ vào tin đã đóng gây trải nghiệm xấu và tổn hại uy tín nền tảng nhiều hơn so với các loại nội dung khác chấp nhận độ trễ index cao hơn.
- Tin tuyển dụng có ngày hết hạn tự động phải được gỡ khỏi index đúng thời điểm hết hạn mà không cần nhà tuyển dụng chủ động thao tác, xử lý đúng múi giờ và không để lệch vài giờ khiến tin hết hạn vẫn hiển thị hoặc tin còn hạn bị gỡ nhầm.
- Tìm kiếm theo mức lương phải xử lý được các tin để khoảng lương (min-max) thay vì một con số cố định, và một số tin không công khai mức lương — bộ lọc phải trả kết quả đúng logic giao khoảng khi ứng viên lọc theo một mức lương mong muốn cụ thể, không loại nhầm các tin có khoảng lương chỉ giao một phần với tiêu chí lọc.
- Kỹ năng yêu cầu trong tin đăng thường được nhà tuyển dụng gõ tự do, trong khi ứng viên tìm theo tên kỹ năng chuẩn — autocomplete và index phải ánh xạ được các cách viết/tên gọi khác nhau của cùng một kỹ năng (viết tắt, đồng nghĩa) về cùng một khái niệm để không bỏ lỡ tin phù hợp chỉ vì khác cách gõ.
- Khi một công ty đăng số lượng lớn tin cùng lúc hoặc gỡ hàng loạt tin cùng lúc, pipeline index phải xử lý được đợt thay đổi hàng loạt này mà không tạo ra độ trễ lan sang việc cập nhật index cho các tin đăng bình thường khác của nhà tuyển dụng khác đang diễn ra song song.
