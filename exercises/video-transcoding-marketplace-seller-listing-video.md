# Marketplace với video sản phẩm tự upload từ seller

**Hệ thống:** Một marketplace cho phép seller tự upload video giới thiệu sản phẩm khi đăng tin bán, chất lượng/định dạng đầu vào rất khác nhau tùy thiết bị quay (điện thoại, máy quay chuyên nghiệp).

**Vai trò của flow:** Nhận video từ seller, chuẩn hóa về định dạng/chất lượng thống nhất để hiển thị trên trang sản phẩm, không làm chậm quá trình đăng tin bán hàng của seller.

**Yêu cầu cụ thể:**
- Video đầu vào có thể ở đủ loại codec, độ phân giải, tỉ lệ khung hình, và cả hướng quay tùy thiết bị seller dùng — pipeline chuẩn hóa phải nhận diện đúng và xử lý được sự đa dạng này thành output nhất quán trên trang sản phẩm mà không yêu cầu seller phải tự chỉnh sửa/convert trước khi upload.
- Việc seller đăng tin bán không nên bị chặn chờ video xử lý xong hoàn toàn — tin sản phẩm phải hiển thị được ngay với ảnh/thông tin khác trong khi video còn đang transcode ở nền, và tự động cập nhật hiển thị video khi xử lý xong mà không cần seller thao tác lại.
- Một số video từ seller cá nhân dùng điện thoại quay ở chất lượng thấp/rung/tối — hệ thống cần có ngưỡng kiểm tra chất lượng đầu vào tối thiểu để từ chối sớm hoặc cảnh báo seller thay vì lãng phí tài nguyên transcode ra một video chất lượng kém không cải thiện được gì so với gốc.
- Khi seller upload video mới để thay thế video cũ của một tin đang bán chạy, việc thay thế phải diễn ra mà không tạo khoảng trống hiển thị (tin tạm thời không có video) hoặc hiển thị nhầm video cũ/video mới lẫn lộn cho các khách đang xem tin cùng lúc.
- Với lượng seller lớn upload đồng thời ở giờ cao điểm đăng tin, hàng đợi transcode phải đảm bảo công bằng giữa các seller (tránh một seller upload nhiều video liên tục chiếm hết tài nguyên khiến seller khác phải chờ lâu bất thường), đồng thời kiểm soát được chi phí xử lý khi khối lượng video từ seller cá nhân thường nhỏ lẻ nhưng số lượng rất lớn.
