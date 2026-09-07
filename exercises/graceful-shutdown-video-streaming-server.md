# Server streaming video cần shutdown mà không cắt ngang luồng đang phát

**Hệ thống:** Một server phát video theo yêu cầu (streaming) cho hàng nghìn client cùng lúc, mỗi client giữ một kết nối HTTP dài (long-lived, có thể kéo dài từ vài phút tới vài giờ tùy độ dài video) để nhận dữ liệu video liên tục.

**Vai trò của flow:** Khác với request ngắn thông thường, draining ở đây không thể chỉ chờ vài chục giây rồi cắt — phải xử lý đúng cách với các kết nối có thể còn kéo dài rất lâu, mà không làm gián đoạn trải nghiệm xem của người dùng đang giữa chừng video.

**Yêu cầu cụ thể:**
- Khi nhận tín hiệu shutdown, instance phải phân loại các kết nối streaming đang mở theo thời lượng còn lại ước tính — với các luồng sắp kết thúc tự nhiên trong vài phút, để chúng tự hoàn tất; với các luồng còn rất dài, phải có chiến lược chuyển giao (ví dụ báo client tự động reconnect sang instance khác tại đúng vị trí đang xem) thay vì chờ vô thời hạn.
- Grace period cho streaming không thể dùng một con số cố định ngắn như service API thông thường — phải định nghĩa theo phân vị của thời lượng video thực tế (ví dụ đủ để 95% luồng đang phát tự kết thúc), và với phần còn lại chấp nhận chủ động ngắt có kiểm soát kèm tín hiệu để client resume đúng vị trí, không để phát hiện lỗi im lặng.
- Đảm bảo khi một kết nối streaming bị buộc ngắt do hết grace period, response phải trả về theo cách client player hiểu được là "cần reconnect" (ví dụ đóng luồng kèm mã trạng thái/tín hiệu rõ ràng), chứ không phải một kết nối bị treo hoặc timeout im lặng khiến player hiển thị lỗi phát khó chịu cho người xem.
- Xử lý trường hợp nhiều instance cùng shutdown gần như đồng thời (ví dụ do autoscale scale-in loạt lớn cùng lúc với deploy) — tránh dồn toàn bộ client đang xem bị buộc reconnect cùng một thời điểm gây spike tải đột biến lên các instance còn lại hoặc lên CDN edge phía trước.
- Theo dõi và cảnh báo riêng tỷ lệ luồng bị ngắt giữa chừng do shutdown (khác với ngắt do người dùng chủ động dừng xem), để phân biệt được mức độ ảnh hưởng thực sự của quá trình draining lên trải nghiệm người dùng, không gộp chung vào số liệu drop-off thông thường.
