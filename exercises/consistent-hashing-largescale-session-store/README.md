# Session store quy mô lớn cho hàng chục triệu user online

**Hệ thống:** Nền tảng lưu session của hàng chục triệu user đang online cùng lúc, phân tán session data ra nhiều node để chịu tải, cần đảm bảo tính sẵn sàng cao vì mất session đồng nghĩa user bị đăng xuất ngoài ý muốn.

**Vai trò của flow:** Consistent hashing quyết định session của một user thuộc node nào, và khi cụm phải scale (thêm/bớt node do thay đổi tải), việc rebalance session chỉ ảnh hưởng một phần nhỏ session, không được làm mất hàng loạt session cùng lúc khi node thay đổi.

**Yêu cầu cụ thể:**
- Khi thêm hoặc bớt node trong cụm lưu session, chỉ những session thuộc vùng hash bị dịch chuyển mới cần di chuyển sang node mới — dùng virtual node đủ nhiều để đảm bảo tải session không dồn lệch lên vài node vật lý ngay sau khi scale.
- Trong lúc đang rebalance (di chuyển session từ node cũ sang node mới), request của user có session đang được di chuyển phải vẫn được phục vụ đúng — cần chiến lược rõ ràng (double-read ở cả node cũ và mới trong giai đoạn chuyển tiếp, hoặc giữ bảng ánh xạ "đang di chuyển" để router biết hỏi node nào) để tránh user bị văng ra ngoài (session not found) giữa chừng dù session thực tế vẫn còn hợp lệ.
- Khi một node lưu session chết đột ngột (không phải scale có kế hoạch mà là sự cố), toàn bộ session trên node đó theo consistent hashing thuần sẽ dồn hết sang node kế tiếp trên ring — nếu không có replication trước đó, đây là mất session hàng loạt cho đúng dải user rơi vào node chết; cần nêu rõ chiến lược replicate session sang N node kế cận trên ring để chịu được việc mất một node mà không mất session.
- Session có TTL (hết hạn tự nhiên) chồng lấn với việc bị di chuyển do rebalance — cần đảm bảo TTL gốc được giữ nguyên khi session chuyển node (không bị reset TTL về đầy do quá trình di chuyển), tránh tình trạng session lẽ ra đã hết hạn tự nhiên nhưng lại "sống lại" đầy đủ TTL sau khi rebalance.
- Đo lường: tỉ lệ session bị lỗi "not found" tạm thời trong lúc scale cụm (session miss rate during rebalance), thời gian hoàn tất rebalance cho một đợt scale, và số lượng session thực sự bị mất (nếu có) sau sự cố node chết để đánh giá độ hiệu quả của chiến lược replication.
