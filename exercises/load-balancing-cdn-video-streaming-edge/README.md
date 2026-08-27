# Load balancing tại edge server cho CDN video streaming

**Hệ thống:** Một hệ thống CDN phục vụ video streaming, có nhiều edge node đặt gần người dùng theo khu vực địa lý, mỗi edge node chịu tải hàng nghìn viewer xem đồng thời.

**Vai trò của flow:** Load balancing ở tầng edge phải chọn được node vừa gần người dùng nhất (giảm độ trễ) vừa còn đủ tải, và xử lý đúng khi một edge node đang phục vụ nhiều viewer bỗng trở nên quá tải giữa lúc đang phát video cho họ.

**Yêu cầu cụ thể:**
- Việc chọn edge node cho một viewer mới phải kết hợp cả yếu tố địa lý (độ trễ mạng ước tính) lẫn tải hiện tại của node (số viewer đang xem, băng thông đang dùng) — không chỉ chọn node gần nhất theo khoảng cách địa lý đơn thuần nếu node đó đã gần đạt giới hạn băng thông.
- Xử lý trường hợp edge node đang phục vụ hàng nghìn viewer bỗng vượt ngưỡng tải (do một sự kiện đông người xem cùng lúc) — với các viewer mới, phải chuyển hướng sang node lân cận khác; với các viewer đang xem dở trên node đó, cần chiến lược giảm tải có kiểm soát (ví dụ hạ chất lượng bitrate một số phiên) thay vì để tất cả cùng bị giật/lag đồng loạt.
- Đảm bảo việc chuyển một viewer từ edge node quá tải sang node khác giữa lúc đang phát không làm gián đoạn luồng phát video mà người dùng nhận biết được — cần cơ chế chuyển tiếp êm (ví dụ player tự động reconnect tới node mới tại đúng thời điểm đang phát) tương tự cơ chế đổi kênh CDN, không phải ngắt và load lại từ đầu.
- Xử lý race condition khi số liệu tải của một edge node được cập nhật từ nhiều nguồn gần như đồng thời (viewer mới join, viewer rời đi, viewer đổi chất lượng bitrate) — quyết định route viewer mới phải dựa trên số liệu đủ mới, tránh dồn nhiều viewer mới vào cùng một node vì đọc phải số liệu tải đã lỗi thời một nhịp.
- Thiết kế cơ chế phát hiện một edge node bị suy giảm chất lượng dịch vụ dù vẫn "healthy" theo health check cơ bản (ví dụ tỷ lệ buffering/rebuffer của viewer trên node đó tăng cao dù node không báo lỗi hệ thống), và giảm dần trọng số route viewer mới tới node đó trước khi tình trạng trở nên nghiêm trọng hơn.
