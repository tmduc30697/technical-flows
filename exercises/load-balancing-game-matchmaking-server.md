# Matchmaking server cho game nhiều người chơi thời gian thực

**Hệ thống:** Một backend game multiplayer real-time, người chơi trong cùng một "room" cần được route tới đúng game server đang giữ trạng thái room đó.

**Vai trò của flow:** Consistent hashing theo `room_id` để đảm bảo toàn bộ player trong một room luôn kết nối tới đúng server instance xử lý logic game của room đó (khác với LB thường route độc lập từng request).

**Yêu cầu cụ thể:**
- Khi một room được tạo, gán cố định vào một server instance qua consistent hashing; mọi player join room sau đó phải được route đúng instance đó dù họ kết nối ở thời điểm khác nhau.
- Xử lý trường hợp server instance giữ room bị crash giữa trận: phát hiện qua health check, và có chiến lược rõ ràng (kết thúc trận hòa/khôi phục từ snapshot state gần nhất) chứ không im lặng drop kết nối.
- Đảm bảo không xảy ra "split-brain": hai server khác nhau đồng thời tin rằng mình đang giữ cùng một room (do stale routing table) — cần cơ chế lease/ownership có TTL.
- Cân bằng tải giữa các server theo số room đang active (không chỉ theo số connection), vì mỗi room tiêu tốn CPU khác nhau tùy số người chơi.
- Viết test mô phỏng scale cluster từ 3 lên 6 instance khi đang có hàng trăm room active, đo số room bị buộc phải di chuyển (rebalance) và đảm bảo player không bị disconnect trong quá trình đó.
