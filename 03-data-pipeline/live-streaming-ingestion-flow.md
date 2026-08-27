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

---

## Livestream bán hàng với chốt đơn tức thời

**Repository:** `live-streaming-live-shopping-checkout-sync`

**Hệ thống:** Một nền tảng livestream bán hàng, người bán giới thiệu sản phẩm trực tiếp, viewer đặt mua ngay trong lúc xem, giá/khuyến mãi/tồn kho có thể thay đổi liên tục ngay trong buổi live.

**Vai trò của flow:** Nhận ingest từ người bán, transcode/phân phối tới viewer, đồng thời đảm bảo các mốc sự kiện chốt giá/mở bán hiển thị trên luồng được đóng dấu thời gian chính xác để hệ thống đặt hàng đối chiếu đúng, bất kể độ trễ phân phối khác nhau giữa các viewer.

**Yêu cầu cụ thể:**
- Độ trễ ingest-tới-viewer khác nhau giữa các viewer (do CDN edge, chất lượng mạng) có thể khiến viewer xem chậm hơn nhìn thấy giá/tồn kho đã lỗi thời tại thời điểm đặt hàng — cần đóng dấu thời gian sự kiện chốt giá/mở bán ngay tại nguồn phát để hệ thống order xác thực theo trạng thái thực tại thời điểm phát sóng, không theo thời điểm viewer bấm mua trên máy của họ.
- Khi người bán bị rớt kết nối đột ngột giữa lúc nhiều viewer đang chuẩn bị chốt đơn, hệ thống phải phân biệt được "gián đoạn tạm thời" và "kết thúc phiên live" để tránh đóng phiên bán hàng quá sớm khiến các đơn đang xử lý dở bị hủy oan.
- Overlay số lượng tồn kho hiển thị trên luồng video luôn có độ trễ vài giây so với hệ thống tồn kho thực — hệ thống đặt hàng không được tin tưởng tuyệt đối vào con số hiển thị trên overlay mà phải luôn đối chiếu lại tồn kho thực tại thời điểm xử lý đơn, đồng thời không để overlay lệch quá xa gây viewer bức xúc khi đặt hàng thất bại.
- Thời điểm mở bán flash sale trong live có thể khiến lượng viewer tăng đột biến trong vài giây, kéo tải lên cả pipeline ingest/CDN lẫn hệ thống đặt hàng downstream — cần cơ chế cách ly để tải đột biến phía checkout không làm ảnh hưởng ngược lại tới độ ổn định của luồng video đang phát cho những viewer khác.
- Khi buổi live được ghi lại làm VOD phát lại sau, các mốc thời gian giới thiệu từng sản phẩm phải được lưu chính xác đồng bộ với luồng gốc, tránh tình huống viewer xem VOD bấm mua vào sản phẩm/giá đã hết hiệu lực kể từ buổi live gốc.

---

## Hội thảo trực tuyến với nhiều người thuyết trình

**Repository:** `live-streaming-webinar-multi-presenter-mixing`

**Hệ thống:** Một nền tảng tổ chức hội thảo trực tuyến với nhiều người thuyết trình, luân phiên chia sẻ màn hình/webcam, người xem theo dõi qua một luồng output duy nhất.

**Vai trò của flow:** Nhận đồng thời nhiều luồng ingest (webcam, chia sẻ màn hình) từ các presenter khác nhau, ghép (mix/switch) thành một luồng output mượt mà, chuyển tiếp real-time tới toàn bộ người xem.

**Yêu cầu cụ thể:**
- Khi quyền trình bày được chuyển từ presenter này sang presenter khác, việc switch nguồn phải diễn ra mượt, không để lộ khung hình đen/đứng hình hoặc lẫn audio của hai nguồn cùng lúc do lệch thời điểm giữa lệnh chuyển và thời điểm luồng ingest thực tế đến.
- Mỗi luồng ingest từ presenter có độ trễ mạng khác nhau (người ở xa datacenter, mạng yếu) — hệ thống mixing phải bù trừ được lệch thời gian giữa các nguồn trước khi ghép, tránh tình trạng audio của một presenter phát ra không khớp với hành động trên màn hình chia sẻ đang hiển thị.
- Nếu một presenter bị rớt kết nối ngay giữa lúc đang là nguồn chính đang phát, hệ thống phải tự động phát hiện và chuyển output sang nguồn còn sống (slide cuối cùng nhận được, hoặc presenter khác) trong thời gian ngắn nhất, không để cả buổi hội thảo đứng hình chờ presenter đó kết nối lại.
- Khi nhiều presenter cùng bật micro cùng lúc do thao tác nhầm sau khi nhường lời, hệ thống ghép audio không được để chồng tiếng làm người nghe không hiểu ai đang nói — cần cơ chế phát hiện/ưu tiên đúng nguồn audio đang được đánh dấu là trình bày chính.
- Việc ghi lại toàn bộ buổi hội thảo để phát lại phải giữ đúng trình tự chuyển đổi giữa các nguồn như đã phát trực tiếp, kể cả các khoảng chuyển tiếp ngắn giữa các presenter, để người xem lại không nhầm lẫn thứ tự trình bày so với thực tế diễn ra.

---

## Lớp học trực tuyến kéo dài nhiều giờ với tương tác hai chiều

**Repository:** `live-streaming-online-classroom-interactive-sync`

**Hệ thống:** Một nền tảng dạy học trực tuyến, giáo viên stream bài giảng trong các buổi học kéo dài nhiều giờ liên tục, học sinh tương tác hai chiều (giơ tay, đặt câu hỏi qua chat) trong lúc học.

**Vai trò của flow:** Nhận luồng bài giảng từ giáo viên, phân phối tới học sinh, đồng thời đảm bảo các sự kiện tương tác được đồng bộ đúng thời điểm với nội dung đang giảng và được ghi lại chính xác phục vụ xem lại sau.

**Yêu cầu cụ thể:**
- Với buổi học kéo dài nhiều giờ liên tục, pipeline ingest/ghi hình phải chịu được vận hành ổn định lâu dài (tránh rò rỉ tài nguyên worker, tích lũy lỗi nhỏ theo thời gian) mà không cần restart giữa chừng làm gián đoạn giờ học, khác hẳn các luồng live ngắn vài chục phút.
- Sự kiện học sinh giơ tay hoặc gửi câu hỏi phải được gắn đúng mốc thời gian tương ứng với nội dung giáo viên đang giảng tại thời điểm đó, kể cả khi độ trễ phân phối khiến học sinh xem chậm hơn giáo viên vài giây — tránh tình huống giáo viên trả lời một câu hỏi khi nội dung bài giảng đã trôi qua xa so với thời điểm học sinh thực sự đặt câu hỏi.
- Nếu kết nối của giáo viên bị gián đoạn giữa buổi giảng, hệ thống phải cho phép khôi phục đúng vị trí trong buổi học mà không tạo phiên mới, đồng thời không làm mất các tương tác đã phát sinh trong lúc gián đoạn.
- Bản ghi lại buổi học để xem sau phải giữ đồng bộ chính xác giữa video bài giảng và các mốc tương tác quan trọng (câu hỏi được giáo viên trả lời, thời điểm chuyển sang phần bài mới), để học sinh xem lại tra cứu đúng theo mốc thời gian mà không bị lệch do độ trễ tích lũy qua nhiều giờ ghi hình.
- Khi số lượng học sinh tham gia đồng thời một lớp lớn (hàng trăm/nghìn), lượng sự kiện tương tác gửi lên gần như đồng thời có thể tạo áp lực ghi nhận lớn — cần đảm bảo kênh tương tác này không cạnh tranh tài nguyên với luồng video chính, tránh làm giảm chất lượng/độ trễ phát sóng khi lớp học sôi động.
