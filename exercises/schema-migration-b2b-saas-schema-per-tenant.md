# Đổi từ multi-tenant chung schema sang schema-per-tenant trong SaaS B2B

**Hệ thống:** Một SaaS quản lý nhân sự B2B đang dùng 1 schema chung với cột `tenant_id` để phân biệt dữ liệu các công ty khách hàng, cần chuyển một số tenant lớn sang schema riêng để cải thiện performance.

**Vai trò của flow:** Với mỗi tenant đang migrate, flow phải đảm bảo request ghi dữ liệu (thêm nhân viên, sửa lương...) trong lúc chuyển đổi vẫn đi đúng vào schema đang là "nguồn sự thật" hiện tại của tenant đó, tránh việc 1 request ghi vào schema cũ sau khi cutover đã hoàn tất.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế "routing table" theo `tenant_id` xác định tenant đang ở schema nào (cũ/đang migrate/đã chuyển xong), và mọi request phải tra bảng này trước khi ghi — không hardcode schema trong code app.
- Mô tả cụ thể tình huống cutover: ngay tại thời điểm chuyển routing của 1 tenant từ schema cũ sang schema mới, có request đang giữa transaction ghi vào schema cũ — yêu cầu cơ chế "khóa ghi tạm thời" (maintenance window ngắn, vài giây) cho riêng tenant đó trong lúc cutover, các request khác của tenant khác không bị ảnh hưởng.
- Trong giai đoạn dual-write trước cutover, mọi write cho tenant đang migrate phải ghi cả 2 schema trong cùng 1 transaction phân tán (hoặc pattern outbox/saga nếu 2 schema ở 2 connection khác nhau), có xử lý rõ khi 1 trong 2 bên ghi thất bại (retry hoặc rollback toàn bộ, không để 1 bên có 1 bên không).
- Yêu cầu đọc dữ liệu trong giai đoạn dual-write luôn đọc từ schema cũ (nguồn sự thật) cho tới khi cutover, để tránh đọc dữ liệu chưa backfill đầy đủ ở schema mới.
- Có checklist rollback: nếu sau cutover phát hiện lỗi dữ liệu ở schema mới, phải trả routing của tenant đó về schema cũ trong vòng vài phút mà không mất giao dịch nào ghi vào schema mới trong khoảng thời gian đã cutover.
