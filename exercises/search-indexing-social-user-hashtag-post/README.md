# Mạng xã hội tìm kiếm người dùng, hashtag và bài viết

**Hệ thống:** Một mạng xã hội cần chức năng tìm kiếm người dùng, hashtag, và nội dung bài viết, kết hợp gợi ý autocomplete cá nhân hóa và trending.

**Vai trò của flow:** Lập chỉ mục liên tục nội dung mới (bài viết, hashtag) và hồ sơ người dùng, phục vụ tìm kiếm kết hợp mức độ liên quan, độ phổ biến, và cá nhân hóa theo người tìm.

**Yêu cầu cụ thể:**
- Bài viết mới đăng phải xuất hiện được trong kết quả tìm kiếm trong thời gian rất ngắn (gần thời gian thực) để phục vụ các sự kiện đang nóng (breaking news, viral moment), khác với yêu cầu index chậm hơn có thể chấp nhận ở catalog sản phẩm tĩnh.
- Autocomplete tìm người dùng phải cá nhân hóa theo mối quan hệ của người tìm (ưu tiên bạn bè/người đang follow lên trước người lạ cùng tên), không chỉ xếp hạng theo độ phổ biến chung.
- Hashtag trending phải được tính toán và cập nhật gần thời gian thực dựa trên tốc độ tăng trưởng sử dụng, không phải chỉ dựa trên tổng số lượt dùng lịch sử (tránh hashtag cũ dùng nhiều nhưng đã hết hot vẫn đứng đầu).
- Bài viết/tài khoản bị xóa hoặc bị khóa do vi phạm chính sách phải biến mất khỏi kết quả tìm kiếm và autocomplete ngay lập tức, đồng bộ với hành động kiểm duyệt nội dung, không có khoảng trễ để nội dung vi phạm vẫn tìm được.
- Index phải chịu được khối lượng ghi rất lớn liên tục (hàng nghìn bài viết/giây ở giờ cao điểm) mà không làm chậm độ trễ đọc (tìm kiếm) của người dùng khác đang truy vấn cùng lúc.
