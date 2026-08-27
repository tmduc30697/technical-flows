# Nền tảng phát sóng thể thao kết hợp cá cược trực tiếp

**Hệ thống:** Một nền tảng phát sóng trận đấu thể thao trực tiếp, đồng thời hiển thị tỷ lệ cá cược cập nhật theo diễn biến trận đấu.

**Vai trò của flow:** Nhận luồng phát sóng gốc từ đơn vị sản xuất, transcode và phân phối tới hàng trăm nghìn người xem đồng thời với yêu cầu độ trễ đồng nhất giữa mọi người xem để đảm bảo công bằng cá cược.

**Yêu cầu cụ thể:**
- Độ trễ từ nguồn đến từng viewer phải càng đồng nhất càng tốt giữa các viewer (không để người dùng CDN edge này xem sớm hơn người khác quá nhiều), vì chênh lệch trực tiếp ảnh hưởng đến công bằng đặt cược.
- Phải có cơ chế đồng bộ giữa luồng video và dữ liệu tỷ lệ cá cược/tỷ số hiển thị overlay, tránh trường hợp overlay hiển thị sự kiện trước khi hình ảnh thực tế diễn ra.
- Hệ thống ingest phải có phương án dự phòng (nguồn phát thứ hai/backup feed) tự động chuyển sang khi nguồn chính gặp sự cố kỹ thuật, không để mất sóng giữa trận đấu quan trọng.
- Hỗ trợ time-shift/DVR ngắn (xem lại vài chục giây gần nhất) mà không ảnh hưởng đến luồng trực tiếp đang phân phối cho người khác.
- Toàn bộ luồng và mốc thời gian sự kiện phải được log đầy đủ để phục vụ đối soát khi có tranh chấp về kết quả cá cược liên quan đến độ trễ hiển thị.
