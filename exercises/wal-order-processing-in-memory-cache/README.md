# WAL backup cho hệ thống xử lý đơn hàng dùng in-memory cache tăng tốc

**Hệ thống:** Hệ thống xử lý đơn hàng giữ trạng thái tạm (đơn đang xử lý, bước hiện tại) trong in-memory cache để tăng tốc độ đọc/ghi thay vì query DB liên tục, nhưng in-memory cache có thể mất sạch khi service restart hoặc crash.

**Vai trò của flow:** WAL ghi mọi thay đổi trạng thái đơn hàng ra đĩa trước hoặc song song với việc cập nhật in-memory cache, để khi cache mất do crash/restart, có thể replay WAL tái tạo lại đúng trạng thái các đơn đang xử lý thay vì mất dấu.

**Yêu cầu cụ thể:**
- Thứ tự bắt buộc: ghi thay đổi vào WAL trước, cập nhật in-memory cache sau — nếu làm ngược lại (cập nhật cache trước, ghi WAL sau) thì một crash xảy ra đúng khoảng giữa hai bước sẽ khiến trạng thái "tưởng đã xử lý" trong cache biến mất mà không có bản ghi nào trong WAL để phục hồi.
- Khi service restart, phải replay WAL để rebuild lại toàn bộ in-memory cache về đúng trạng thái trước lúc crash trước khi bắt đầu nhận request xử lý đơn mới; nếu chấp nhận request trong lúc cache còn rỗng/chưa replay xong sẽ gây đọc sai trạng thái đơn hàng (ví dụ tưởng đơn chưa xử lý bước nào trong khi thực tế đã xử lý gần xong).
- Nhiều đơn hàng đang ở các bước khác nhau tại thời điểm crash (một số vừa ghi WAL xong nhưng cache chưa kịp cập nhật, một số đã cập nhật cache nhưng chưa tới đợt flush/checkpoint) — replay phải xử lý đúng từng đơn theo entry cuối cùng ghi nhận cho đơn đó trong WAL, không áp dụng nhầm thứ tự giữa các đơn khác nhau.
- Tần suất thay đổi trạng thái đơn hàng có thể rất cao (nhiều bước nhỏ mỗi đơn) khiến ghi WAL đồng bộ mỗi thay đổi ảnh hưởng tới lợi ích tốc độ mà in-memory cache mang lại — cần cân nhắc rõ giữa ghi WAL đồng bộ từng bước (an toàn tuyệt đối, chậm hơn) và batch/ghi định kỳ (nhanh hơn, chấp nhận mất một khoảng nhỏ thay đổi gần nhất nếu crash), và phải nêu rõ RPO chấp nhận được cho hệ thống đơn hàng này.
- Có cơ chế phát hiện WAL và in-memory cache bị lệch nhau (ví dụ do bug ghi cache thất bại âm thầm dù WAL đã ghi đúng) thông qua kiểm tra định kỳ hoặc checksum trạng thái tổng hợp, tránh để hệ thống chạy lâu dài với cache sai lệch mà không ai phát hiện cho tới khi có sự cố lớn mới lộ ra.
