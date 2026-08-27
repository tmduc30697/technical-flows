# Nền tảng gaming với ingest đa vùng địa lý

**Hệ thống:** Một nền tảng streaming tập trung vào game thủ, có người xem và streamer phân bố ở nhiều khu vực địa lý khác nhau trên thế giới.

**Vai trò của flow:** Định tuyến luồng ingest của streamer tới điểm nhận (point of presence) gần nhất về địa lý, xử lý và phân phối lại toàn cầu, đảm bảo trải nghiệm ổn định dù streamer/viewer ở đâu.

**Yêu cầu cụ thể:**
- Streamer phải được tự động định tuyến tới ingest endpoint gần nhất dựa trên vị trí địa lý/độ trễ mạng đo được, không ép cứng một endpoint trung tâm duy nhất.
- Nếu điểm ingest gần nhất gặp sự cố (quá tải, downtime khu vực), phải tự động failover sang điểm ingest khác mà không làm gián đoạn quá vài giây hoặc yêu cầu streamer cấu hình lại thủ công.
- Nội dung sau khi ingest ở một khu vực phải được nhân bản/phân phối tới các khu vực khác để viewer ở xa vẫn xem được với độ trễ hợp lý, không phải kéo trực tiếp từ điểm ingest gốc.
- Xử lý được trường hợp streamer di chuyển (đổi mạng, đổi vị trí giữa buổi live) mà điểm ingest tối ưu thay đổi — không được làm rớt stream khi chuyển đổi điểm ingest ngầm.
- Theo dõi được chi phí băng thông theo từng khu vực địa lý để tối ưu vận hành, cảnh báo khi một khu vực có chi phí bất thường so với số viewer thực tế.
