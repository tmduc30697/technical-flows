# Live streaming ingestion flow — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — nền tảng streaming giải trí, livestream bán hàng, hội thảo trực tuyến, phát sóng thể thao, nền tảng gaming đa vùng, và lớp học trực tuyến — nhằm luyện đủ các góc của flow tiếp nhận luồng live (ingest, transcode real-time, phân phối CDN, độ trễ, khả năng chịu lỗi).

---

## Nền tảng streaming giải trí kiểu Twitch

**Repository:** `live-streaming-entertainment-twitch-like`

**Hệ thống:** Một nền tảng cho phép streamer phát trực tiếp gameplay/nội dung giải trí tới đông đảo người xem.

**Vai trò của flow:** Nhận luồng RTMP/SRT từ phần mềm phát của streamer, transcode real-time ra nhiều bitrate và phân phối qua CDN tới người xem.

**Yêu cầu cụ thể:**
- Ingest server phải xác thực stream key trước khi nhận luồng, từ chối ngay các kết nối dùng key sai/đã bị revoke, không cho tạo luồng "ma".
- Transcode real-time ra tối thiểu 3 mức bitrate (nguồn, trung, thấp) để hỗ trợ adaptive bitrate streaming, độ trễ thêm vào không vượt quá vài giây so với luồng gốc.
- Nếu kết nối ingest từ streamer bị rớt giữa buổi (mất mạng), hệ thống phải phát hiện trong vòng vài giây và tự động chuyển stream sang trạng thái "gián đoạn" hiển thị cho viewer, tự khôi phục khi streamer kết nối lại mà không tạo stream session mới.
- Toàn bộ segment video trong lúc live phải được lưu tạm để phục vụ ghép VOD sau khi kết thúc, không được để mất segment khi có sự cố worker transcode.
- Kiến trúc phân phối phải chịu được lượng viewer đồng thời lớn tăng đột biến (raid, streamer nổi tiếng lên live) mà không làm tăng độ trễ cho toàn bộ viewer khác.

---

## Nền tảng phát sóng thể thao kết hợp cá cược trực tiếp

**Repository:** `live-streaming-sports-betting`

**Hệ thống:** Một nền tảng phát sóng trận đấu thể thao trực tiếp, đồng thời hiển thị tỷ lệ cá cược cập nhật theo diễn biến trận đấu.

**Vai trò của flow:** Nhận luồng phát sóng gốc từ đơn vị sản xuất, transcode và phân phối tới hàng trăm nghìn người xem đồng thời với yêu cầu độ trễ đồng nhất giữa mọi người xem để đảm bảo công bằng cá cược.

**Yêu cầu cụ thể:**
- Độ trễ từ nguồn đến từng viewer phải càng đồng nhất càng tốt giữa các viewer (không để người dùng CDN edge này xem sớm hơn người khác quá nhiều), vì chênh lệch trực tiếp ảnh hưởng đến công bằng đặt cược.
- Phải có cơ chế đồng bộ giữa luồng video và dữ liệu tỷ lệ cá cược/tỷ số hiển thị overlay, tránh trường hợp overlay hiển thị sự kiện trước khi hình ảnh thực tế diễn ra.
- Hệ thống ingest phải có phương án dự phòng (nguồn phát thứ hai/backup feed) tự động chuyển sang khi nguồn chính gặp sự cố kỹ thuật, không để mất sóng giữa trận đấu quan trọng.
- Hỗ trợ time-shift/DVR ngắn (xem lại vài chục giây gần nhất) mà không ảnh hưởng đến luồng trực tiếp đang phân phối cho người khác.
- Toàn bộ luồng và mốc thời gian sự kiện phải được log đầy đủ để phục vụ đối soát khi có tranh chấp về kết quả cá cược liên quan đến độ trễ hiển thị.

---

## Nền tảng gaming với ingest đa vùng địa lý

**Repository:** `live-streaming-gaming-multi-region-ingest`

**Hệ thống:** Một nền tảng streaming tập trung vào game thủ, có người xem và streamer phân bố ở nhiều khu vực địa lý khác nhau trên thế giới.

**Vai trò của flow:** Định tuyến luồng ingest của streamer tới điểm nhận (point of presence) gần nhất về địa lý, xử lý và phân phối lại toàn cầu, đảm bảo trải nghiệm ổn định dù streamer/viewer ở đâu.

**Yêu cầu cụ thể:**
- Streamer phải được tự động định tuyến tới ingest endpoint gần nhất dựa trên vị trí địa lý/độ trễ mạng đo được, không ép cứng một endpoint trung tâm duy nhất.
- Nếu điểm ingest gần nhất gặp sự cố (quá tải, downtime khu vực), phải tự động failover sang điểm ingest khác mà không làm gián đoạn quá vài giây hoặc yêu cầu streamer cấu hình lại thủ công.
- Nội dung sau khi ingest ở một khu vực phải được nhân bản/phân phối tới các khu vực khác để viewer ở xa vẫn xem được với độ trễ hợp lý, không phải kéo trực tiếp từ điểm ingest gốc.
- Xử lý được trường hợp streamer di chuyển (đổi mạng, đổi vị trí giữa buổi live) mà điểm ingest tối ưu thay đổi — không được làm rớt stream khi chuyển đổi điểm ingest ngầm.
- Theo dõi được chi phí băng thông theo từng khu vực địa lý để tối ưu vận hành, cảnh báo khi một khu vực có chi phí bất thường so với số viewer thực tế.
