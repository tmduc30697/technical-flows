# Hội thảo trực tuyến với nhiều người thuyết trình

**Hệ thống:** Một nền tảng tổ chức hội thảo trực tuyến với nhiều người thuyết trình, luân phiên chia sẻ màn hình/webcam, người xem theo dõi qua một luồng output duy nhất.

**Vai trò của flow:** Nhận đồng thời nhiều luồng ingest (webcam, chia sẻ màn hình) từ các presenter khác nhau, ghép (mix/switch) thành một luồng output mượt mà, chuyển tiếp real-time tới toàn bộ người xem.

**Yêu cầu cụ thể:**
- Khi quyền trình bày được chuyển từ presenter này sang presenter khác, việc switch nguồn phải diễn ra mượt, không để lộ khung hình đen/đứng hình hoặc lẫn audio của hai nguồn cùng lúc do lệch thời điểm giữa lệnh chuyển và thời điểm luồng ingest thực tế đến.
- Mỗi luồng ingest từ presenter có độ trễ mạng khác nhau (người ở xa datacenter, mạng yếu) — hệ thống mixing phải bù trừ được lệch thời gian giữa các nguồn trước khi ghép, tránh tình trạng audio của một presenter phát ra không khớp với hành động trên màn hình chia sẻ đang hiển thị.
- Nếu một presenter bị rớt kết nối ngay giữa lúc đang là nguồn chính đang phát, hệ thống phải tự động phát hiện và chuyển output sang nguồn còn sống (slide cuối cùng nhận được, hoặc presenter khác) trong thời gian ngắn nhất, không để cả buổi hội thảo đứng hình chờ presenter đó kết nối lại.
- Khi nhiều presenter cùng bật micro cùng lúc do thao tác nhầm sau khi nhường lời, hệ thống ghép audio không được để chồng tiếng làm người nghe không hiểu ai đang nói — cần cơ chế phát hiện/ưu tiên đúng nguồn audio đang được đánh dấu là trình bày chính.
- Việc ghi lại toàn bộ buổi hội thảo để phát lại phải giữ đúng trình tự chuyển đổi giữa các nguồn như đã phát trực tiếp, kể cả các khoảng chuyển tiếp ngắn giữa các presenter, để người xem lại không nhầm lẫn thứ tự trình bày so với thực tế diễn ra.
