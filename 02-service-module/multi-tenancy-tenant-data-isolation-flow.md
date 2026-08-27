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

---

## Cách ly dữ liệu bán hàng giữa các seller trên marketplace dùng chung nền tảng

**Repository:** `multi-tenancy-marketplace-seller-isolation`

**Hệ thống:** Một marketplace cho phép nhiều người bán (seller) độc lập đăng sản phẩm và bán hàng trên cùng một nền tảng dùng chung hạ tầng, mỗi seller có dữ liệu đơn hàng, doanh thu, và thông tin khách hàng của riêng họ.

**Vai trò của flow:** Cách ly ở đây phải đảm bảo một seller không bao giờ nhìn thấy dữ liệu bán hàng/doanh thu của seller khác dù họ đang cùng thao tác trên chung một giao diện quản lý (dashboard người bán) và chung hạ tầng backend, khác với mô hình một công ty một tenant thông thường vì số lượng seller có thể lên tới hàng chục nghìn và liên tục có seller mới tham gia/rời đi.

**Yêu cầu cụ thể:**
- Mọi truy vấn liên quan tới đơn hàng, doanh thu, tồn kho trên dashboard người bán phải bị giới hạn đúng theo `seller_id` ở tầng thấp nhất (không dựa vào việc ẩn nút bấm trên giao diện), kể cả với các API nội bộ dùng chung để tính toán số liệu tổng hợp cho nhiều mục đích khác nhau của nền tảng.
- Xử lý đúng trường hợp một đơn hàng có sản phẩm từ nhiều seller khác nhau trong cùng một giỏ hàng của khách mua — mỗi seller chỉ được xem phần đơn hàng liên quan tới sản phẩm của mình (thông tin vận chuyển, số lượng, giá của phần họ bán), không được thấy toàn bộ chi tiết đơn hàng gộp bao gồm phần của seller khác.
- Đảm bảo các báo cáo/thống kê xếp hạng do nền tảng cung cấp cho seller (ví dụ so sánh hiệu suất bán hàng với "mức trung bình ngành") không bị suy luận ngược ra số liệu cụ thể của một seller đối thủ cụ thể — đặc biệt khi ngành hàng đó chỉ có rất ít seller tham gia khiến số liệu trung bình gần như lộ số liệu của một seller cụ thể.
- Xử lý trường hợp một seller ngừng hoạt động/bị khóa tài khoản (vi phạm chính sách hoặc tự nguyện rời sàn) — dữ liệu lịch sử đơn hàng liên quan tới khách mua vẫn cần được giữ cho mục đích hỗ trợ khách hàng và pháp lý, nhưng seller đã rời sàn không được tiếp tục truy cập vào dữ liệu đó, và các seller khác cũng không được vô tình thấy dữ liệu của seller đã rời sàn qua các báo cáo tổng hợp toàn nền tảng.
- Đảm bảo tính năng chăm sóc khách hàng của nền tảng (nhân viên marketplace hỗ trợ khách mua) khi cần tra cứu một đơn hàng cụ thể chỉ được xem đúng phạm vi dữ liệu liên quan tới đơn hàng đó, không được có quyền truy cập mặc định vào toàn bộ dữ liệu kinh doanh của seller liên quan, và mọi lần tra cứu phải được ghi log lại.

---

## Cách ly dữ liệu giữa các phòng ban dùng chung một công cụ nội bộ

**Repository:** `multi-tenancy-internal-tool-department-isolation`

**Hệ thống:** Một công cụ nội bộ (ví dụ hệ thống quản lý tài liệu/quy trình phê duyệt) được một công ty triển khai dùng chung cho nhiều phòng ban (nhân sự, tài chính, kỹ thuật, kinh doanh), trong đó mỗi phòng ban đóng vai trò như một "tenant" dù tất cả đều thuộc cùng một công ty.

**Vai trò của flow:** Khác với mô hình tenant=công ty khách hàng thông thường, ở đây ranh giới cách ly nằm trong nội bộ một tổ chức — độ khó nằm ở chỗ nhân viên vẫn cần một số dữ liệu dùng chung toàn công ty (thư mục nhân viên, thông báo chung) trong khi dữ liệu nghiệp vụ riêng của từng phòng ban (hồ sơ nhân sự, số liệu tài chính nội bộ) phải được cách ly chặt giữa các phòng, và một người có thể thuộc nhiều phòng ban hoặc đổi phòng ban theo thời gian.

**Yêu cầu cụ thể:**
- Thiết kế mô hình phân quyền phân biệt rõ ba loại tài nguyên: tài nguyên cách ly tuyệt đối theo phòng ban (không phòng ban nào khác được xem), tài nguyên dùng chung toàn công ty, và tài nguyên chia sẻ có chọn lọc giữa một số phòng ban cụ thể (dự án liên phòng ban) — mỗi loại cần cơ chế kiểm tra quyền khác nhau, không dùng chung một quy tắc cách ly cứng nhắc như mô hình tenant=công ty độc lập.
- Xử lý trường hợp một nhân viên đổi phòng ban (chuyển từ phòng kỹ thuật sang phòng kinh doanh) — quyền truy cập dữ liệu của phòng ban cũ phải được thu hồi đúng thời điểm chuyển đổi có hiệu lực, trong khi dữ liệu họ đã tạo ra khi còn ở phòng cũ (tài liệu, phê duyệt đã ký) vẫn cần giữ nguyên thuộc về phòng ban cũ cho mục đích lưu trữ, không tự động chuyển theo người.
- Đảm bảo vai trò quản trị viên hệ thống (admin IT phụ trách vận hành công cụ nội bộ) có quyền vận hành kỹ thuật (backup, cấu hình) nhưng không mặc định có quyền đọc nội dung dữ liệu nghiệp vụ nhạy cảm của từng phòng ban (ví dụ hồ sơ lương của phòng nhân sự) — tách bạch quyền vận hành hệ thống khỏi quyền truy cập nội dung nghiệp vụ.
- Xử lý đúng trường hợp dự án liên phòng ban cần chia sẻ có chọn lọc một phần dữ liệu giữa hai hoặc nhiều phòng ban trong một khoảng thời gian nhất định — quyền chia sẻ này phải có thời hạn rõ ràng và tự động thu hồi khi dự án kết thúc, không để việc chia sẻ tạm thời trở thành lỗ hổng truy cập vĩnh viễn giữa các phòng ban do quên thu hồi quyền.
- Đảm bảo tính năng tìm kiếm/gợi ý toàn công cụ (ví dụ tìm kiếm tài liệu, autocomplete) không vô tình làm lộ sự tồn tại của dữ liệu thuộc phòng ban khác qua kết quả gợi ý hoặc số lượng kết quả tìm kiếm trả về, dù nội dung chi tiết vẫn bị chặn — vì ngay cả việc biết "có một tài liệu tên X tồn tại" cũng có thể là rò rỉ thông tin nhạy cảm giữa các phòng ban đang cạnh tranh nội bộ (ví dụ tài liệu tái cơ cấu).
