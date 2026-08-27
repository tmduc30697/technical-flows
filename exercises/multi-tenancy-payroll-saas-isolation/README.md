# Cách ly dữ liệu nhân sự/lương giữa các công ty khách hàng trên SaaS payroll

**Hệ thống:** Một nền tảng SaaS quản lý lương và nhân sự cho nhiều công ty khách hàng, mỗi công ty (tenant) có dữ liệu cực nhạy cảm (lương, thông tin cá nhân nhân viên, số tài khoản ngân hàng).

**Vai trò của flow:** Ngoài cách ly giữa các tenant (công ty), flow còn phải xử lý cách ly trong nội bộ một tenant (một nhân viên không được xem lương của nhân viên khác, một quản lý phòng ban chỉ xem được nhân viên phòng mình).

**Yêu cầu cụ thể:**
- Thiết kế phân quyền hai lớp: lớp ngoài cách ly tuyệt đối giữa các công ty khách hàng khác nhau (tenant), lớp trong cách ly theo vai trò trong nội bộ một công ty (nhân viên thường, quản lý phòng ban, HR, admin) — cả hai lớp đều phải được kiểm tra ở mọi endpoint truy cập dữ liệu lương/nhân sự.
- Xử lý đúng trường hợp một người dùng có vai trò khác nhau ở các phòng ban khác nhau trong cùng một công ty (ví dụ vừa là quản lý phòng A vừa là nhân viên thường của một dự án liên phòng ban) — quyền xem dữ liệu phải được tính đúng theo ngữ cảnh cụ thể, không cấp nhầm quyền cao nhất cho mọi ngữ cảnh.
- Đảm bảo các báo cáo tổng hợp (ví dụ báo cáo chi phí lương toàn công ty cho CFO) không bị rò rỉ thông tin chi tiết từng cá nhân qua việc suy luận gián tiếp (ví dụ một phòng ban chỉ có 1 người, báo cáo "lương trung bình phòng ban" vô tình để lộ đúng lương của người đó).
- Khi một công ty khách hàng offboard nhân viên của họ (nhân viên nghỉ việc), dữ liệu của nhân viên đó vẫn phải được giữ cách ly đúng theo tenant (công ty) đó cho mục đích lưu trữ hồ sơ theo quy định pháp luật lao động, không bị gộp lẫn hoặc mất ranh giới tenant theo thời gian.
- Ghi log chi tiết và có thể audit mọi lần truy cập vào dữ liệu lương cụ thể của một nhân viên (ai xem, khi nào, thuộc công ty nào xem dữ liệu công ty nào) để phục vụ điều tra khi có khiếu nại về việc lộ thông tin lương.
