# Event ticket queueing flow (chống bot/scalping) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều loại hệ thống bán vé sự kiện (concert lớn, thể thao, hội nghị công nghệ, sự kiện giới hạn số lượng cực nhỏ, bán vé combo nhiều buổi) để luyện thiết kế hàng đợi ảo (virtual waiting room), chống bot mua vét vé, và xử lý đúng đắn số lượng vé giới hạn dưới traffic cực lớn tại đúng giờ mở bán.

---

## Hàng đợi ảo cho concert lớn với hàng trăm nghìn người chờ cùng lúc

**Repository:** `ticket-queue-concert-virtual-waiting-room`

**Hệ thống:** Một nền tảng bán vé concert của idol nổi tiếng, dự kiến hàng trăm nghìn người truy cập cùng lúc tại giờ mở bán cho vài chục nghìn vé.

**Vai trò của flow:** Flow phải xếp người dùng vào hàng đợi ảo theo đúng thứ tự truy cập, chỉ cho một lượng giới hạn người vào trang chọn vé/thanh toán mỗi lúc để hệ thống backend không bị sập, đồng thời phải công bằng (không cho phép gian lận vị trí hàng đợi).

**Yêu cầu cụ thể:**
- Vị trí trong hàng đợi phải được xác định dựa trên thời điểm request đến server (không dựa vào thời điểm client tự ghi nhận, vì có thể bị giả mạo bởi script tự động refresh liên tục để giữ vị trí đẹp).
- Mô tả cụ thể cơ chế chống gian lận: nếu 1 người mở nhiều tab/nhiều thiết bị để lấy nhiều vị trí hàng đợi khác nhau cho cùng 1 mục đích mua vé, hệ thống nên có cơ chế giới hạn theo user đã đăng nhập (chỉ 1 vị trí hàng đợi hiệu lực/user) thay vì cho phép nhiều vị trí song song từ cùng 1 người.
- Khi đến lượt vào trang chọn vé, người dùng có 1 khoảng thời gian giới hạn để hoàn tất chọn vé và thanh toán (ví dụ 10 phút); nếu hết thời gian phải tự động đưa họ ra khỏi lượt và không được tự động cấp lại lượt mới ngay (phải chờ xếp hàng lại hoặc quy định rõ chính sách retry), tránh 1 người chiếm dụng slot xử lý quá lâu làm chậm những người sau.
- Việc cấp "lượt vào" (token cho phép truy cập trang mua vé) phải là atomic tăng dần theo đúng công suất xử lý backend cho phép tại một thời điểm (ví dụ tối đa 5000 người đang trong bước mua cùng lúc), không cấp vượt quá công suất dù hàng đợi ảo có bao nhiêu người đang chờ.
- Có cơ chế theo dõi phát hiện hành vi bất thường (một địa chỉ IP/device fingerprint tạo hàng trăm session hàng đợi trong thời gian ngắn) để đánh dấu nghi vấn bot, không chặn cứng ngay (tránh chặn nhầm người dùng mạng chia sẻ) nhưng có thể yêu cầu xác thực bổ sung (captcha) trước khi cấp lượt vào mua vé.

---

## Bán vé thể thao (bóng đá) với hạn chế mỗi CMND/CCCD chỉ mua tối đa N vé

**Repository:** `ticket-queue-sports-id-purchase-limit`

**Hệ thống:** Một hệ thống bán vé xem bóng đá, để chống phe vé, mỗi CMND/CCCD chỉ được mua tối đa 4 vé cho 1 trận, xác thực qua số CCCD khi mua.

**Vai trò của flow:** Flow phải validate giới hạn số vé theo CCCD một cách atomic dưới tải cao, chống việc dùng nhiều tài khoản đăng nhập khác nhau nhưng cùng 1 CCCD để lách giới hạn.

**Yêu cầu cụ thể:**
- Giới hạn "tối đa 4 vé/CCCD/trận" phải được kiểm tra và cập nhật atomic dựa trên số CCCD (không phải theo tài khoản đăng nhập), dùng unique constraint hoặc bảng đếm riêng theo (event_id, cccd_number) để 2 tài khoản khác nhau nhưng khai báo cùng 1 CCCD không thể cùng mua vượt tổng 4 vé.
- Mô tả cụ thể race: 2 tài khoản khác nhau (cùng 1 người dùng 2 tài khoản, khai cùng CCCD) gửi request mua 3 vé và 2 vé gần như đồng thời (tổng 5 vé, vượt giới hạn 4) — chỉ có tổ hợp không vượt giới hạn được xác nhận theo transaction atomic tới trước dựa trên tổng đã mua thực tế tại thời điểm mỗi request xử lý, request vượt giới hạn phải bị từ chối rõ ràng dựa trên số đã mua cập nhật mới nhất (không dựa vào số liệu đọc từ trước khi request kia xử lý xong).
- Khi hệ thống xác thực CCCD (gọi tới cơ sở dữ liệu dân cư hoặc dịch vụ xác thực bên thứ ba) có độ trễ, việc "giữ chỗ" số lượng vé tạm thời trong lúc chờ xác thực phải có TTL ngắn và atomic, để không giữ vé treo vô thời hạn nếu bước xác thực bị timeout hoặc thất bại.
- Nếu 1 CCCD đã mua đủ 4 vé nhưng sau đó 1 giao dịch bị hủy (hoàn tiền vé), phải cập nhật atomic để CCCD đó có thể mua lại vé mới trong giới hạn còn lại, và số vé đã hủy phải được trả về pool tồn kho chung ngay.
- Có cơ chế rate limit theo CCCD/thiết bị bổ sung cho tầng API (ngoài giới hạn ở tầng transaction) để giảm số lượng request thử nghiệm lách luật (ví dụ dò nhiều tổ hợp số lượng vé khác nhau) chạm tới transaction chính, giảm tải không cần thiết cho DB.

---

## Bán vé hội nghị công nghệ với early-bird pricing thay đổi theo số lượng đã bán

**Repository:** `ticket-queue-tech-conference-early-bird`

**Hệ thống:** Một hội nghị công nghệ bán vé theo tier giá tăng dần: giá early-bird cho 200 vé đầu, sau đó tăng giá cho tier tiếp theo, lặp lại cho tới khi hết vé.

**Vai trò của flow:** Flow phải xác định đúng giá áp dụng cho mỗi giao dịch mua dựa trên số vé đã bán tại đúng thời điểm giao dịch, xử lý đúng khi nhiều người mua cùng lúc gần ranh giới chuyển tier giá.

**Yêu cầu cụ thể:**
- Việc trừ tồn kho vé và xác định tier giá áp dụng phải nằm trong cùng 1 transaction atomic: kiểm tra số vé đã bán, xác định tier, trừ tồn kho theo đúng tier đó — không tách thành 2 bước riêng (đọc số đã bán để hiển thị giá, rồi charge theo giá đã hiển thị mà không re-check).
- Mô tả cụ thể: đã bán 199 vé early-bird, 3 người mua cùng lúc 1 vé mỗi người (người mua vé thứ 200, 201, 202) — chỉ đúng người mua vé thứ 200 được tính giá early-bird, người thứ 201 và 202 phải được tính đúng giá tier tiếp theo, dựa trên transaction atomic xác định đúng thứ tự xử lý thực tế tại DB (không phải thứ tự client gửi request theo timestamp hiển thị trên UI).
- Nếu 1 người mua nhiều vé trong 1 giao dịch (ví dụ mua 5 vé) mà số lượng đó vắt qua ranh giới tier (199 vé early-bird còn lại nhưng người này mua 5 vé, tức 1 vé early-bird + 4 vé tier tiếp theo), quy định rõ chính sách: có cho phép chia giá hỗn hợp trong 1 giao dịch hay không, và nếu có, phải tính đúng atomic cả 2 mức giá trong cùng 1 transaction.
- Trang hiển thị "còn bao nhiêu vé ở tier hiện tại" phải phản ánh đúng số liệu real-time đã trừ đi các giao dịch đang xử lý (không chỉ các giao dịch đã hoàn tất), để tránh hiển thị sai khiến nhiều người cùng nhắm vào tưởng còn early-bird trong khi thực ra đã hết.
- Khi 1 giao dịch mua vé early-bird thất bại ở bước thanh toán (sau khi đã tạm trừ tồn kho tier), phải hoàn trả đúng atomic vào đúng tier đã trừ (không trả nhầm vào tier hiện tại nếu tier đã chuyển đổi trong lúc xử lý), để không làm sai lệch số lượng vé còn lại của tier gốc.

---

## Bán vé sự kiện siêu giới hạn số lượng cực nhỏ (buổi gặp fan riêng, 20 chỗ)

**Repository:** `ticket-queue-fan-meetup-ultra-limited`

**Hệ thống:** Một nền tảng bán vé cho buổi gặp fan riêng cực kỳ giới hạn (chỉ 20 chỗ) của 1 celebrity, nhu cầu vượt xa cung nên gần như chắc chắn có tranh chấp gay gắt ngay giây đầu mở bán.

**Vai trò của flow:** Flow phải xử lý chính xác tuyệt đối khi số lượng cực nhỏ (20) đối mặt với lượng request có thể lên tới hàng chục nghìn trong giây đầu tiên, đảm bảo đúng 20 người mua được, không hơn không kém, và công bằng về thứ tự.

**Yêu cầu cụ thể:**
- Vì số lượng cực nhỏ và tranh chấp cực cao, khuyến nghị dùng cơ chế lottery (rút số ngẫu nhiên có seed công khai kiểm chứng được) trong 1 khung thời gian đăng ký ngắn, thay vì "ai bấm nhanh hơn thắng" (dễ thiên vị người có kết nối mạng nhanh/dùng script) — mô tả rõ cách sinh và công bố seed để đảm bảo minh bạch, người tham gia có thể tự kiểm tra kết quả không bị can thiệp.
- Nếu vẫn chọn cơ chế "đến trước được trước", phải đảm bảo transaction trừ tồn kho là atomic tuyệt đối bằng update nguyên tử có điều kiện, và có test cụ thể giả lập 20.000 request đồng thời để xác nhận đúng chính xác 20 request thành công, không có sai số dù chỉ 1 vé (over/under selling).
- Mô tả cụ thể trường hợp cần xử lý riêng: 1 trong 20 người thắng bị phát hiện vi phạm điều kiện tham gia (ví dụ dùng nhiều tài khoản) sau khi đã công bố kết quả — quy định quy trình hủy vé của người đó và chuyển đúng cho người kế tiếp trong danh sách dự phòng (nếu dùng lottery, cần giữ danh sách dự phòng theo đúng thứ tự rút số ban đầu, không rút số lại từ đầu).
- Toàn bộ quy trình (đăng ký, xử lý, công bố kết quả) phải có log/audit trail đầy đủ và không thể chỉnh sửa ngược (immutable) để chống khiếu nại/nghi ngờ gian lận từ phía fan, đặc biệt quan trọng với sự kiện có giá trị cảm xúc/tranh chấp cao.
- Có kế hoạch xử lý tải hệ thống cho khung thời gian đăng ký (dù không "bán" theo kiểu FCFS, vẫn có thể có traffic dồn vào đúng lúc mở đăng ký) tương tự các kỹ thuật hàng đợi ảo, dù bản chất bên trong là random.

---

## Bán vé combo nhiều buổi cho festival kéo dài nhiều ngày

**Repository:** `ticket-queue-festival-multi-day-combo`

**Hệ thống:** Một festival âm nhạc kéo dài 3 ngày, bán vé combo (đủ 3 ngày) và vé từng ngày riêng lẻ, mỗi loại vé có tồn kho riêng nhưng đều liên quan tới tổng số lượng khách tối đa cho phép vào khu vực festival mỗi ngày.

**Vai trò của flow:** Flow phải quản lý tồn kho vé combo và vé lẻ sao cho tổng số khách mỗi ngày (tính cả người dùng vé combo và vé lẻ ngày đó) không vượt giới hạn sức chứa khu vực, dưới tải mua đồng thời cao ở giờ mở bán.

**Yêu cầu cụ thể:**
- Sức chứa mỗi ngày phải được tính là 1 pool chung mà cả vé combo và vé lẻ ngày đó cùng trừ vào, không quản lý 2 tồn kho độc lập rồi cộng dồn ở bước hiển thị (dễ dẫn tới bán vượt sức chứa thực nếu chỉ kiểm tra riêng từng loại vé).
- Mô tả cụ thể: sức chứa ngày 2 còn 100 chỗ, có người mua vé combo (chiếm 1 chỗ ngày 2) và người mua vé lẻ ngày 2 (chiếm 1 chỗ ngày 2) gửi request gần như đồng thời với tổng vượt quá 100 — cả 2 loại giao dịch phải cùng đi qua 1 update atomic trừ vào đúng pool sức chứa ngày 2 chung, không xử lý ở 2 transaction riêng biệt không biết về nhau.
- Khi mua vé combo, phải trừ atomic đồng thời cả 3 pool sức chứa (ngày 1, 2, 3) trong 1 transaction duy nhất — nếu chỉ còn đủ chỗ cho 2 trong 3 ngày (ví dụ ngày 3 đã hết chỗ), toàn bộ giao dịch mua combo phải fail hoàn toàn, không được bán combo thiếu 1 ngày.
- Nếu 1 người đổi vé combo thành vé lẻ (hủy 1-2 ngày, giữ lại 1 ngày) trước sự kiện, phải trả lại atomic đúng số chỗ vào các pool sức chứa của những ngày bị hủy, để những chỗ đó có thể bán tiếp cho người khác đang muốn mua vé lẻ ngày đó.
- Trang hiển thị tình trạng vé phải phân biệt rõ và cập nhật đúng real-time cho từng loại (còn combo/còn vé lẻ ngày 1/2/3), phản ánh đúng ảnh hưởng qua lại giữa các loại vé (ví dụ khi combo bán hết vì 1 ngày cụ thể hết chỗ, vé lẻ những ngày còn dư chỗ vẫn phải hiển thị đúng là còn bán được).
