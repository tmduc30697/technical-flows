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

## Livestream bán hàng (live shopping) trên nền tảng e-commerce

**Repository:** `live-streaming-live-shopping`

**Hệ thống:** Một tính năng trong app e-commerce cho phép seller livestream giới thiệu sản phẩm, gắn link mua ngay trong lúc live.

**Vai trò của flow:** Nhận luồng live từ seller và phân phối tới người xem với độ trễ thấp, đồng bộ với các sự kiện tương tác (chat, chèn sản phẩm, chốt đơn) diễn ra real-time.

**Yêu cầu cụ thể:**
- Độ trễ từ lúc seller nói đến lúc viewer thấy/nghe phải đủ thấp để tương tác mua hàng theo thời gian thực có ý nghĩa (ví dụ dưới vài giây), khác với yêu cầu độ trễ của video giải trí thông thường.
- Sự kiện "chèn sản phẩm lên màn hình" và "chốt đơn số lượng giới hạn" phải được đồng bộ thời gian chính xác với luồng video, không bị lệch so với những gì seller đang nói trên hình.
- Khi lượng viewer tăng vọt do sản phẩm hot (flash sale trong live), CDN/edge phân phối phải scale ra kịp mà không làm rớt kết nối của viewer đang xem.
- Nếu luồng ingest bị gián đoạn ngắn, phải có cơ chế hiển thị lại đúng trạng thái đơn hàng/giỏ hàng cho viewer sau khi luồng khôi phục, không làm mất giao dịch đang thực hiện.
- Ghi lại toàn bộ luồng và log sự kiện tương tác để dùng cho việc tính hoa hồng, xử lý tranh chấp đơn hàng phát sinh trong lúc live sau này.

---

## Nền tảng hội thảo trực tuyến/webinar cho doanh nghiệp

**Repository:** `live-streaming-enterprise-webinar`

**Hệ thống:** Một SaaS B2B cho phép tổ chức hội thảo trực tuyến với nhiều presenter, hàng trăm/nghìn người tham dự.

**Vai trò của flow:** Nhận nhiều luồng ingest từ các presenter khác nhau, ghép/chuyển cảnh (switch) giữa các luồng, và phân phối một luồng thống nhất tới người tham dự, đồng thời ghi lại để phát lại sau.

**Yêu cầu cụ thể:**
- Hỗ trợ nhận đồng thời nhiều luồng ingest từ các presenter (mỗi người một kết nối riêng) và cho phép host chuyển đổi (switch) presenter nào đang được phát chính, không làm giật/đen màn hình phía người xem khi chuyển.
- Presenter bị rớt kết nối giữa buổi phải được phát hiện và tự động chuyển sang presenter/màn hình chờ khác, không làm dừng toàn bộ webinar.
- Ghi lại toàn bộ buổi hội thảo (bao gồm cả các lần chuyển presenter) thành một file VOD liên tục, đúng thứ tự, để publish lại cho người không tham dự được.
- Giới hạn số người tham dự đồng thời theo gói cước của tổ chức; hệ thống phải từ chối người tham dự vượt hạn mức một cách rõ ràng (hàng đợi chờ hoặc thông báo), không làm giảm chất lượng cho người đã vào.
- Có cơ chế đo và báo cáo độ trễ/chất lượng luồng theo thời gian thực để đội vận hành phát hiện sự cố trước khi người dùng phàn nàn.

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

---

## Lớp học trực tuyến (EdTech) phát trực tiếp

**Repository:** `live-streaming-edtech-online-class`

**Hệ thống:** Một nền tảng edtech cho giáo viên dạy trực tiếp qua video, học sinh tham gia và tương tác qua chat/tay giơ (raise hand).

**Vai trò của flow:** Nhận luồng live từ giáo viên (và đôi khi từ học sinh khi được gọi phát biểu), phân phối tới lớp học, đồng thời ghi lại buổi học và theo dõi điểm danh dựa trên thời gian tham gia.

**Yêu cầu cụ thể:**
- Hỗ trợ chuyển đổi giữa luồng chính của giáo viên và luồng của học sinh được gọi lên phát biểu, không làm gián đoạn luồng chính quá lâu khi chuyển đổi qua lại nhiều lần trong một buổi học.
- Ghi nhận chính xác thời điểm vào/ra của từng học sinh trong luồng live để tính điểm danh, xử lý đúng trường hợp học sinh bị rớt mạng và vào lại nhiều lần (không tính trùng, không tính thiếu thời gian tham gia).
- Buổi học phải được ghi lại đầy đủ thành VOD ngay sau khi kết thúc để học sinh vắng mặt xem lại, kèm join đúng với danh sách điểm danh đã ghi nhận.
- Chất lượng luồng phải tự điều chỉnh (giảm bitrate) khi băng thông giáo viên/học sinh yếu, ưu tiên giữ luồng không bị đứng hình hơn là giữ độ phân giải cao.
- Giới hạn số lớp học live đồng thời theo gói cước của trường/tổ chức, từ chối rõ ràng khi vượt hạn mức thay vì để tất cả các lớp cùng bị giảm chất lượng.
