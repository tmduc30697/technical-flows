# Multi-tenancy/tenant data isolation flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (SaaS B2B dùng chung schema, nền tảng hồ sơ y tế cần compliance cao, marketplace nhiều người bán, SaaS nhân sự/payroll, công cụ nội bộ đa phòng ban) để luyện việc cách ly dữ liệu giữa các tenant đúng đắn và không để lộ dữ liệu chéo tenant do lỗi logic.

---

## Cách ly dữ liệu theo tenant dùng chung schema/database cho SaaS quản lý dự án

**Repository:** `multi-tenancy-project-management-shared-schema`

**Hệ thống:** Một nền tảng SaaS quản lý dự án cho nhiều công ty khách hàng (tenant), dùng chung một database và schema, phân biệt dữ liệu bằng cột `tenant_id` trên mỗi bảng.

**Vai trò của flow:** Đây là mô hình multi-tenancy phổ biến nhất (shared schema) — flow phải đảm bảo mọi truy vấn dữ liệu đều bị giới hạn đúng theo tenant của người dùng đang thao tác, không có đường nào truy cập nhầm sang dữ liệu tenant khác.

**Yêu cầu cụ thể:**
- Mọi query đọc/ghi dữ liệu phải bắt buộc có điều kiện lọc theo `tenant_id` ở tầng thấp nhất có thể (ví dụ qua middleware/ORM tự động thêm điều kiện, hoặc row-level security ở database) — không phụ thuộc vào việc lập trình viên nhớ thêm điều kiện này ở từng câu query riêng lẻ, vì chỉ cần quên một chỗ là có thể lộ dữ liệu chéo tenant.
- Viết test tự động (không chỉ manual test) cố tình thử truy cập dữ liệu của tenant B bằng token/session của tenant A qua mọi endpoint quan trọng, đảm bảo luôn bị chặn (403/404) không phải chỉ dựa vào việc UI không hiển thị link dẫn tới đó.
- Xử lý đúng các tài nguyên dùng chung giữa nhiều tenant nhưng cần cấu hình riêng (ví dụ file upload, cấu hình thông báo) — đảm bảo đường dẫn lưu trữ và khóa truy cập (như signed URL) đều gắn chặt với tenant, không dùng ID tài nguyên có thể đoán được để truy cập chéo tenant.
- Xử lý trường hợp một nhân viên hệ thống (support/admin nội bộ) cần truy cập dữ liệu của một tenant cụ thể để hỗ trợ — phải qua một cơ chế impersonation/xem hộ có kiểm soát, ghi log đầy đủ, không dùng chung quyền truy cập không giới hạn tenant cho mục đích vận hành.
- Đảm bảo các thao tác nền (background job, cron, cache) cũng tôn trọng ranh giới tenant — ví dụ một job tính báo cáo tổng hợp chạy sai logic gộp nhầm dữ liệu của nhiều tenant vào một kết quả chung phải được phát hiện qua test, không chỉ API request trực tiếp mới cần kiểm tra.

---

## Cách ly nghiêm ngặt cho nền tảng hồ sơ sức khỏe đa phòng khám

**Repository:** `multi-tenancy-healthcare-strict-isolation`

**Hệ thống:** Một nền tảng SaaS cho nhiều phòng khám tư nhân (mỗi phòng khám là một tenant) quản lý hồ sơ bệnh nhân, chịu yêu cầu bảo mật/compliance dữ liệu y tế cao hơn nhiều so với SaaS thông thường.

**Vai trò của flow:** Do mức độ nhạy cảm và hậu quả pháp lý nếu lộ dữ liệu chéo tenant (dữ liệu bệnh nhân của phòng khám A lộ sang phòng khám B), flow cách ly ở đây cần nhiều lớp bảo vệ hơn shared schema thông thường.

**Yêu cầu cụ thể:**
- Đánh giá và lựa chọn mô hình cách ly phù hợp với mức độ nhạy cảm (ví dụ database riêng theo tenant hoặc schema riêng theo tenant, thay vì chỉ cột `tenant_id` trong shared schema) — giải thích rõ trade-off giữa mức độ cách ly cao hơn và độ phức tạp vận hành/chi phí tăng thêm.
- Đảm bảo khóa mã hóa dữ liệu (encryption key) tại rest là riêng biệt theo từng tenant, để nếu một khóa bị lộ (hoặc một tenant yêu cầu xóa toàn bộ dữ liệu và hủy khóa — crypto shredding) không ảnh hưởng tới dữ liệu của các tenant khác.
- Xử lý yêu cầu xóa dữ liệu hoàn toàn của một tenant (khi phòng khám ngừng hợp tác, theo quy định pháp lý về quyền xóa dữ liệu) — phải xóa/hủy được toàn bộ dữ liệu liên quan trên mọi hệ thống lưu trữ (database chính, cache, backup, log) trong một khung thời gian xác định, có xác nhận hoàn tất.
- Thiết kế audit log chi tiết cho mọi lần truy cập vào hồ sơ bệnh nhân (ai xem, khi nào, dữ liệu nào), tách biệt audit log theo tenant và không cho tenant khác (dù là admin hệ thống ở mức thông thường) xem được audit log của tenant khác.
- Đảm bảo các tính năng dùng AI/phân tích dữ liệu tổng hợp (nếu có, ví dụ thống kê xu hướng bệnh) chỉ được huấn luyện/tính toán trên dữ liệu đã ẩn danh hóa đúng cách và không bao giờ để mô hình học được cách "nhớ" và lộ lại thông tin cụ thể của một bệnh nhân cho tenant khác truy vấn.

---

## Cách ly dữ liệu nhân sự/lương giữa các công ty khách hàng trên SaaS payroll

**Repository:** `multi-tenancy-payroll-saas-isolation`

**Hệ thống:** Một nền tảng SaaS quản lý lương và nhân sự cho nhiều công ty khách hàng, mỗi công ty (tenant) có dữ liệu cực nhạy cảm (lương, thông tin cá nhân nhân viên, số tài khoản ngân hàng).

**Vai trò của flow:** Ngoài cách ly giữa các tenant (công ty), flow còn phải xử lý cách ly trong nội bộ một tenant (một nhân viên không được xem lương của nhân viên khác, một quản lý phòng ban chỉ xem được nhân viên phòng mình).

**Yêu cầu cụ thể:**
- Thiết kế phân quyền hai lớp: lớp ngoài cách ly tuyệt đối giữa các công ty khách hàng khác nhau (tenant), lớp trong cách ly theo vai trò trong nội bộ một công ty (nhân viên thường, quản lý phòng ban, HR, admin) — cả hai lớp đều phải được kiểm tra ở mọi endpoint truy cập dữ liệu lương/nhân sự.
- Xử lý đúng trường hợp một người dùng có vai trò khác nhau ở các phòng ban khác nhau trong cùng một công ty (ví dụ vừa là quản lý phòng A vừa là nhân viên thường của một dự án liên phòng ban) — quyền xem dữ liệu phải được tính đúng theo ngữ cảnh cụ thể, không cấp nhầm quyền cao nhất cho mọi ngữ cảnh.
- Đảm bảo các báo cáo tổng hợp (ví dụ báo cáo chi phí lương toàn công ty cho CFO) không bị rò rỉ thông tin chi tiết từng cá nhân qua việc suy luận gián tiếp (ví dụ một phòng ban chỉ có 1 người, báo cáo "lương trung bình phòng ban" vô tình để lộ đúng lương của người đó).
- Khi một công ty khách hàng offboard nhân viên của họ (nhân viên nghỉ việc), dữ liệu của nhân viên đó vẫn phải được giữ cách ly đúng theo tenant (công ty) đó cho mục đích lưu trữ hồ sơ theo quy định pháp luật lao động, không bị gộp lẫn hoặc mất ranh giới tenant theo thời gian.
- Ghi log chi tiết và có thể audit mọi lần truy cập vào dữ liệu lương cụ thể của một nhân viên (ai xem, khi nào, thuộc công ty nào xem dữ liệu công ty nào) để phục vụ điều tra khi có khiếu nại về việc lộ thông tin lương.
