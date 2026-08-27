# Nền tảng streaming giới hạn số thiết bị xem đồng thời trên một tài khoản

**Hệ thống:** Một nền tảng streaming video trả phí, gói cước giới hạn số thiết bị được phép xem đồng thời (ví dụ tối đa N thiết bị/slot cùng lúc).

**Vai trò của flow:** Quản lý "device slot" đang chiếm quyền xem đồng thời, xử lý đúng khi thiết bị cũ không chủ động logout mà bị thiết bị mới thay thế, và hạn chế chia sẻ tài khoản vượt quá phạm vi cho phép của gói cước.

**Yêu cầu cụ thể:**
- Khi hai thiết bị mới cùng cố giành slot cuối cùng gần như đồng thời (ví dụ hai người trong nhà cùng bấm play khi chỉ còn một slot trống), hệ thống phải đảm bảo không cho cả hai cùng chiếm slot vượt giới hạn, đồng thời cơ chế khóa dùng để xử lý race condition này không được làm chậm đáng kể trải nghiệm bắt đầu phát video.
- Khi thiết bị cũ bị "đá" ra do hết slot vì thiết bị mới giành chỗ, phải dừng phát ngay trên thiết bị bị đá kèm thông báo rõ lý do trên màn hình, không để người dùng gặp phải một lỗi timeout khó hiểu không rõ nguyên nhân.
- Xác định một thiết bị đã "ngừng hoạt động" để tự động giải phóng slot là bài toán khó vì thường không có tín hiệu logout rõ ràng (app bị kill nền, mất mạng đột ngột, đóng tab trình duyệt) — cần cơ chế heartbeat/timeout hợp lý để tránh vừa giữ slot ảo cho thiết bị đã tắt thật sự, vừa tránh giải phóng nhầm slot của một thiết bị chỉ đang mất kết nối tạm thời.
- Cần phân biệt giữa việc cùng một tài khoản được xem trên nhiều thiết bị của một hộ gia đình hợp lệ, với dấu hiệu chia sẻ tài khoản vượt phạm vi cho phép (ví dụ các thiết bị đăng nhập từ nhiều thành phố/quốc gia khác nhau trong cùng khung giờ), để áp dụng chính sách slot chặt hơn cho trường hợp nghi ngờ mà không cản trở người dùng hợp lệ.
- Khi người dùng chủ động quản lý danh sách thiết bị đang chiếm slot và force-logout một thiết bị cụ thể, thao tác này phải giải phóng slot gần như ngay lập tức để thiết bị khác vào xem được, không để người dùng phải chờ hoặc thử lại nhiều lần.
