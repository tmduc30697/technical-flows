# Disaster recovery/failover flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (fintech multi-region, e-commerce database failover, SaaS multi-tenant DR, streaming CDN origin failover, hệ thống hồ sơ y tế cần compliance) để luyện việc thiết kế chuyển đổi dự phòng khi có sự cố lớn mà không mất dữ liệu hoặc gây gián đoạn kéo dài.

---

## Failover đa vùng cho hệ thống xử lý giao dịch fintech

**Repository:** `disaster-recovery-fintech-multi-region-failover`

**Hệ thống:** Một hệ thống xử lý giao dịch chuyển tiền triển khai ở region chính và region phụ (đứng sẵn/standby) ở một khu vực địa lý khác để phòng trường hợp region chính gặp sự cố (mất điện, thiên tai, lỗi hạ tầng cloud provider).

**Vai trò của flow:** Failover flow đảm bảo khi region chính không còn hoạt động, hệ thống chuyển sang region phụ trong thời gian ngắn nhất có thể mà không làm mất hoặc trùng lặp giao dịch tài chính.

**Yêu cầu cụ thể:**
- Định nghĩa rõ và đo được RTO (thời gian tối đa được phép để hệ thống hoạt động trở lại) và RPO (lượng dữ liệu tối đa được phép mất) cho hệ thống này, và thiết kế cơ chế đồng bộ dữ liệu giữa hai region đáp ứng đúng RPO đã cam kết.
- Xử lý trường hợp một giao dịch đang xử lý (đã ghi log ở region chính nhưng chưa kịp đồng bộ sang region phụ) đúng lúc region chính sập — sau khi failover, phải có cơ chế xác định giao dịch đó đã hoàn tất hay chưa (không được để trạng thái "không rõ", vì với tiền bạc phải xác định dứt điểm) và xử lý đúng (hoàn tất tiếp, hoặc rollback và hoàn tiền).
- Đảm bảo tuyệt đối không xử lý trùng lặp một giao dịch hai lần do cả region chính (đang hồi phục một phần) và region phụ (đã failover) cùng cố xử lý — cần cơ chế xác định rõ region nào đang là "nguồn sự thật" (source of truth) tại một thời điểm, tránh split-brain.
- Thiết kế quy trình failback (khi region chính hồi phục, chuyển lại hoạt động chính về đó) an toàn — không tự động failback ngay khi region chính vừa online trở lại mà phải qua bước xác minh đồng bộ dữ liệu đầy đủ và kiểm tra sức khỏe ổn định trong một khoảng thời gian.
- Định kỳ thực hiện diễn tập failover (chaos drill) trên hệ thống thật hoặc staging tương đương, đo lường RTO/RPO thực tế đạt được so với mục tiêu đã cam kết, và ghi nhận các điểm cần cải thiện.

---

## Failover database cho sàn e-commerce khi primary database gặp sự cố

**Repository:** `disaster-recovery-ecommerce-database-failover`

**Hệ thống:** Một sàn e-commerce dùng một database chính (primary) cho việc đọc/ghi và các replica cho việc đọc, cần chuyển đổi tự động sang một replica được nâng cấp thành primary mới khi primary gốc gặp sự cố.

**Vai trò của flow:** Failover flow đảm bảo website tiếp tục hoạt động (ít nhất là các chức năng đọc, và ghi nếu có thể) khi database chính không còn phản hồi, giảm thiểu thời gian downtime nhìn thấy bởi khách hàng.

**Yêu cầu cụ thể:**
- Phải có cơ chế phát hiện chính xác database chính đã thực sự gặp sự cố (không phải chỉ mạng chập chờn tạm thời) trước khi kích hoạt failover, tránh failover không cần thiết gây gián đoạn (ví dụ dựa trên nhiều lần health check thất bại liên tiếp từ nhiều nguồn giám sát độc lập, không chỉ một lần thất bại).
- Trong lúc chưa hoàn tất failover, ứng dụng phải xử lý được ở mức graceful — ví dụ tạm chuyển sang chế độ chỉ đọc (dùng replica cho các trang xem sản phẩm) và báo lỗi rõ ràng cho các hành động cần ghi (đặt hàng) thay vì để toàn site treo/lỗi 500 không rõ nguyên nhân.
- Đảm bảo replica được chọn để promote thành primary mới có độ trễ đồng bộ (replication lag) thấp nhất tại thời điểm sự cố, và có cơ chế xác nhận không mất giao dịch đã commit trước đó (đặc biệt các đơn hàng vừa đặt ngay trước lúc sự cố).
- Sau khi promote một replica thành primary mới, các service khác trong hệ thống (không chỉ tầng database) phải được thông báo và tự động kết nối lại tới primary mới, không cần can thiệp thủ công vào từng service, và không giữ kết nối cache tới primary cũ đã chết.
- Thiết kế test mô phỏng: kill primary database đột ngột trong lúc có traffic đọc/ghi liên tục, đo được thời gian từ lúc sự cố tới lúc hệ thống hoạt động bình thường trở lại, và verify không có đơn hàng nào bị mất dữ liệu (đã xác nhận với khách nhưng không tồn tại trong DB sau failover).

---

## Kế hoạch DR cho nền tảng SaaS multi-tenant khi mất toàn bộ một trung tâm dữ liệu

**Repository:** `disaster-recovery-saas-datacenter-loss`

**Hệ thống:** Một nền tảng SaaS B2B phục vụ hàng trăm tenant (công ty khách hàng) từ một trung tâm dữ liệu chính, cần kế hoạch phục hồi khi toàn bộ trung tâm dữ liệu đó gặp sự cố nghiêm trọng (không chỉ một server/database đơn lẻ).

**Vai trò của flow:** DR flow ở quy mô này phải xử lý việc khôi phục toàn bộ stack (nhiều service, database, hàng đợi, storage) một cách có thứ tự, không chỉ một thành phần.

**Yêu cầu cụ thể:**
- Xác định và document rõ thứ tự khôi phục các thành phần hệ thống (ví dụ database trước, sau đó message queue, sau đó các service phụ thuộc) — không được khôi phục ngẫu nhiên gây các service phụ thuộc khởi động trước khi dependency của chúng sẵn sàng, dẫn tới crash loop.
- Với các tenant có yêu cầu SLA khác nhau (một số tenant trả phí cao hơn có cam kết uptime cao hơn), phải có khả năng ưu tiên khôi phục dịch vụ cho các tenant đó trước nếu không thể khôi phục toàn bộ ngay lập tức.
- Đảm bảo dữ liệu backup được lưu ở một vị trí địa lý hoàn toàn tách biệt khỏi trung tâm dữ liệu chính (không cùng nhà cung cấp điện/mạng) để backup không bị ảnh hưởng bởi chính sự cố đã làm sập trung tâm dữ liệu chính.
- Thiết kế cơ chế thông báo minh bạch cho khách hàng (qua status page độc lập, không phụ thuộc vào hạ tầng đang gặp sự cố) về tình trạng sự cố và tiến độ khôi phục theo thời gian thực, tránh im lặng gây hoang mang cho khách hàng doanh nghiệp.
- Định kỳ (ví dụ mỗi quý) thực hiện diễn tập khôi phục toàn bộ hệ thống từ backup trên một môi trường tách biệt để xác nhận backup thực sự dùng được và đo thời gian khôi phục thực tế, không chỉ tin tưởng vào việc "đã có backup" mà chưa từng thử khôi phục.

---

## Failover CDN origin cho dịch vụ streaming khi origin server chính gặp sự cố

**Repository:** `disaster-recovery-streaming-cdn-origin-failover`

**Hệ thống:** Một dịch vụ streaming video dùng CDN để phân phối nội dung, CDN lấy dữ liệu gốc từ origin server chính, có origin server phụ dự phòng ở một khu vực khác.

**Vai trò của flow:** Failover ở đây phải xử lý đặc thù của nội dung media (file lớn, có thể đang stream giữa chừng) khác với dữ liệu API thông thường.

**Yêu cầu cụ thể:**
- CDN phải tự động chuyển sang lấy dữ liệu từ origin phụ khi origin chính không phản hồi trong một ngưỡng thời gian xác định, và cơ chế phát hiện phải đủ nhanh để không làm người xem chờ đợi lâu nhưng cũng không quá nhạy gây chuyển đổi qua lại không cần thiết khi origin chính chỉ chậm tạm thời.
- Đảm bảo origin phụ có đầy đủ nội dung cần thiết (đồng bộ trước đó) — xử lý rõ trường hợp một nội dung mới vừa upload lên origin chính nhưng chưa kịp đồng bộ sang origin phụ trước khi origin chính sập, và định nghĩa rõ hành vi khi CDN yêu cầu nội dung không tồn tại ở origin phụ (báo lỗi rõ ràng, không trả về nội dung sai hoặc treo vô thời hạn).
- Với người xem đang stream giữa chừng đúng lúc failover xảy ra, player phía client phải có khả năng phát hiện gián đoạn và tự động thử lại/reconnect (dùng cùng vị trí phát hiện được từ trước) mà không bắt người xem phải tự bấm play lại từ đầu.
- Đảm bảo việc chuyển đổi origin không làm lộ ra sự khác biệt về chất lượng/định dạng nội dung giữa hai origin (ví dụ origin phụ có thể chưa transcode đủ các độ phân giải) — cần đồng bộ đầy đủ pipeline xử lý media, không chỉ file gốc.
- Thiết kế giám sát riêng cho tình trạng đồng bộ giữa hai origin (độ trễ đồng bộ, tỷ lệ nội dung đã đồng bộ đầy đủ) để biết trước mức độ sẵn sàng thực tế của origin phụ trước khi cần dùng thật, tránh phát hiện thiếu sót ngay giữa lúc sự cố đang xảy ra.

---

## DR cho hệ thống hồ sơ y tế điện tử với yêu cầu compliance nghiêm ngặt

**Repository:** `disaster-recovery-healthcare-ehr-compliance`

**Hệ thống:** Một hệ thống lưu trữ hồ sơ y tế điện tử cho các cơ sở y tế, chịu quy định pháp lý nghiêm ngặt về bảo mật và tính khả dụng của dữ liệu (không được mất, không được truy cập trái phép cả trong tình huống sự cố).

**Vai trò của flow:** DR ở đây phải cân bằng giữa việc đảm bảo dịch vụ khả dụng (bác sĩ cần truy cập hồ sơ khẩn cấp) và tuân thủ nghiêm ngặt các yêu cầu bảo mật/quyền riêng tư dữ liệu y tế trong suốt quá trình failover.

**Yêu cầu cụ thể:**
- Dữ liệu backup/replica dùng cho DR phải được mã hóa với cùng mức độ bảo mật như dữ liệu chính (không được hạ chuẩn bảo mật để tăng tốc độ đồng bộ), và quyền truy cập vào hệ thống DR phải được kiểm soát chặt tương đương hệ thống chính, không tạo ra "cửa hậu" ít được giám sát hơn.
- Trong tình huống cần truy cập khẩn cấp hồ sơ bệnh nhân khi hệ thống chính không khả dụng (ví dụ cấp cứu), phải có quy trình "break-the-glass access" — cho phép truy cập vượt qua các bước xác thực thông thường trong trường hợp khẩn cấp, nhưng phải ghi log đầy đủ và yêu cầu giải trình sau đó, không được bỏ qua audit trail.
- Đảm bảo trong lúc failover, không có khoảng thời gian nào dữ liệu bệnh nhân bị lộ ra do cấu hình bảo mật tạm thời bị nới lỏng để "cho nhanh khôi phục dịch vụ" — mọi thay đổi cấu hình trong DR đều phải qua review bảo mật, dù đang trong tình huống khẩn cấp.
- Thiết kế cơ chế xác minh tính toàn vẹn dữ liệu sau khi khôi phục từ backup (checksum, đối chiếu số lượng bản ghi) trước khi cho phép hệ thống DR chính thức phục vụ, tránh phục hồi dữ liệu bị hỏng/thiếu mà không phát hiện, gây hậu quả nghiêm trọng cho việc chăm sóc bệnh nhân.
- Ghi lại đầy đủ báo cáo sự cố (incident report) theo yêu cầu quy định ngành y tế sau mỗi lần DR thực sự được kích hoạt, bao gồm thời gian downtime, dữ liệu bị ảnh hưởng (nếu có), và các bước đã thực hiện, phục vụ nghĩa vụ báo cáo cho cơ quan quản lý.
