# Hệ sinh thái nhiều app tiêu dùng dùng chung một tài khoản đăng nhập kiểu Google

**Hệ thống:** Một công ty vận hành nhiều app tiêu dùng riêng biệt (ví dụ app email, app lưu trữ file, app thanh toán, app bản đồ) mà người dùng cuối dùng chung một tài khoản để đăng nhập vào tất cả.

**Vai trò của flow:** Nền tảng tự làm Identity Provider trung tâm cho toàn bộ hệ sinh thái, quản lý một danh tính người dùng nhưng vai trò/quyền truy cập khác nhau ở từng app, và phải chịu tải xác thực ở quy mô rất lớn.

**Yêu cầu cụ thể:**
- Một người dùng có thể có vai trò khác nhau hoàn toàn ở từng app, ví dụ là admin ở app lưu trữ file của tổ chức họ nhưng chỉ là user thường ở app bản đồ — token SSO trung tâm không thể mang theo toàn bộ quyền của mọi app, mỗi app cần tự truy vấn quyền riêng của nó sau khi xác thực danh tính thành công, tránh token phình to hoặc lộ thông tin quyền không liên quan.
- Khi người dùng đổi thông tin nhạy cảm ở tài khoản trung tâm (đổi email, đổi mật khẩu, bật thêm MFA), mọi app con đang có session hoạt động phải nhận được tín hiệu để yêu cầu xác thực lại ở mức phù hợp, tránh trường hợp kẻ tấn công vừa chiếm được tài khoản trung tâm vẫn tiếp tục dùng session cũ ở các app khác vô thời hạn.
- Người dùng có thể dùng cùng một tài khoản cho cả mục đích cá nhân và mục đích qua một tổ chức (ví dụ tài khoản cá nhân nhưng cũng là thành viên workspace công ty) — cần phân tách rõ ngữ cảnh đăng nhập cá nhân/tổ chức để tránh nhầm lẫn quyền, đặc biệt khi chính sách bảo mật giữa hai ngữ cảnh khác nhau đáng kể.
- Ở quy mô hệ sinh thái lớn, một app con bị lỗi hoặc quá tải khi gọi về IdP trung tâm để xác thực không được phép làm sập khả năng đăng nhập của toàn bộ hệ sinh thái — cần cách ly lỗi và có chiến lược cache/fallback hợp lý cho việc xác thực mà không hy sinh khả năng revoke quyền kịp thời khi cần.
- Khi người dùng thu hồi quyền truy cập của một app cụ thể trong hệ sinh thái mà không muốn ảnh hưởng các app khác, hệ thống phải hỗ trợ revoke theo phạm vi từng app riêng lẻ, không cần đăng xuất toàn bộ hệ sinh thái chỉ vì muốn ngắt kết nối một app.
