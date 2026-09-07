# Session store cần nhất quán cao hơn cho hành động nhạy cảm (thanh toán)

**Hệ thống:** Một platform thương mại điện tử lưu session/trạng thái checkout trên store phân tán, đa số truy cập chỉ đọc nhưng bước thanh toán cần chính xác tuyệt đối.

**Vai trò của flow:** Cho phép "tunable consistency" theo API cụ thể — phần lớn request dùng eventual consistency cho tốc độ, riêng bước xác nhận thanh toán chuyển sang strong consistency (CP) để tránh double-charge hay thấy trạng thái sai.

**Yêu cầu cụ thể:**
- Xác định rõ danh sách API nào chạy ở chế độ AP (đọc nhanh, có thể stale) và API nào buộc phải CP (ví dụ xác nhận đã thanh toán, khóa order để tránh 2 request cùng xử lý).
- Với API CP, khi xảy ra network partition và không đủ quorum, hệ thống phải từ chối rõ ràng (trả lỗi "tạm không xử lý được") thay vì âm thầm trả kết quả sai/không chắc chắn.
- Với API AP, phải có cơ chế phát hiện và cảnh báo khi độ "stale" vượt ngưỡng bất thường (ví dụ do một node bị treo lâu) để vận hành can thiệp.
- Test cụ thể case: 2 request thanh toán trùng gửi tới 2 node khác nhau trong lúc network chập chờn — phải đảm bảo chỉ một request thực sự được xác nhận thành công.
- Viết rõ tài liệu cho team khác (API consumer) biết mức consistency guarantee của từng endpoint để họ dùng đúng cách, tránh giả định sai.
