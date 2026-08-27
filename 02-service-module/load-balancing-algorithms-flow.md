# Load balancing algorithms flow (round-robin/least-connection/consistent hashing ở layer LB) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (API gateway cho SaaS B2B, CDN video streaming, fintech, game server, e-commerce flash sale) để luyện việc chọn và triển khai đúng thuật toán load balancing cho từng loại traffic.

---

## Load balancer cho hệ thống thanh toán fintech cần session affinity

**Repository:** `load-balancing-fintech-session-affinity`

**Hệ thống:** Một hệ thống xử lý giao dịch chuyển tiền, trong đó một giao dịch có thể gồm nhiều bước gọi lại cùng service instance (ví dụ giữ state tạm trong bộ nhớ instance để xử lý multi-step OTP).

**Vai trò của flow:** Dùng consistent hashing (hash theo transaction_id hoặc user_id) để đảm bảo các request thuộc cùng một giao dịch luôn được route tới cùng một instance, tránh phải đọc lại state từ DB mỗi bước.

**Yêu cầu cụ thể:**
- Implement consistent hashing ring, hash key là `transaction_id`; đảm bảo cùng key luôn map tới cùng instance trong suốt vòng đời giao dịch (thường vài phút).
- Khi một instance chết giữa lúc đang giữ state của giao dịch đang chạy, phải có cơ chế phát hiện và fail-over request tiếp theo sang instance khác kèm khôi phục state từ nguồn bền (DB/cache) — không được để giao dịch "treo" vô thời hạn.
- Khi thêm/bớt instance khỏi ring (scale in/out), số lượng key phải bị remap là tối thiểu (đúng đặc tính consistent hashing), verify bằng test đo tỷ lệ % key bị đổi instance trước/sau khi thêm 1 node.
- Có cơ chế virtual node (nhiều điểm hash cho một instance) để tránh phân bố lệch tải khi cluster nhỏ (ví dụ chỉ 3-4 instance).
- Đảm bảo bảo mật: hash key không được lộ ra thông tin nhạy cảm qua timing attack (thời gian route không được khác biệt đáng kể giữa các user).

---

## Matchmaking server cho game nhiều người chơi thời gian thực

**Repository:** `load-balancing-game-matchmaking-server`

**Hệ thống:** Một backend game multiplayer real-time, người chơi trong cùng một "room" cần được route tới đúng game server đang giữ trạng thái room đó.

**Vai trò của flow:** Consistent hashing theo `room_id` để đảm bảo toàn bộ player trong một room luôn kết nối tới đúng server instance xử lý logic game của room đó (khác với LB thường route độc lập từng request).

**Yêu cầu cụ thể:**
- Khi một room được tạo, gán cố định vào một server instance qua consistent hashing; mọi player join room sau đó phải được route đúng instance đó dù họ kết nối ở thời điểm khác nhau.
- Xử lý trường hợp server instance giữ room bị crash giữa trận: phát hiện qua health check, và có chiến lược rõ ràng (kết thúc trận hòa/khôi phục từ snapshot state gần nhất) chứ không im lặng drop kết nối.
- Đảm bảo không xảy ra "split-brain": hai server khác nhau đồng thời tin rằng mình đang giữ cùng một room (do stale routing table) — cần cơ chế lease/ownership có TTL.
- Cân bằng tải giữa các server theo số room đang active (không chỉ theo số connection), vì mỗi room tiêu tốn CPU khác nhau tùy số người chơi.
- Viết test mô phỏng scale cluster từ 3 lên 6 instance khi đang có hàng trăm room active, đo số room bị buộc phải di chuyển (rebalance) và đảm bảo player không bị disconnect trong quá trình đó.

---

## E-commerce chịu tải đột biến trong flash sale

**Repository:** `load-balancing-ecommerce-flash-sale-spike`

**Hệ thống:** Một sàn e-commerce có checkout service, trong sự kiện flash sale traffic có thể tăng 20 lần trong vài giây.

**Vai trò của flow:** Load balancer phải tự chuyển đổi hành vi giữa các thuật toán (round-robin lúc bình thường, least-connection khi có instance chạy job nặng) và bảo vệ backend khỏi bị sập toàn cụm.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế phát hiện một instance "chậm" (latency tăng cao) dù vẫn healthy theo health check thông thường, và giảm trọng số route tới instance đó (outlier detection), tránh trường hợp health check pass nhưng response time đã tệ.
- Khi toàn bộ cluster đạt ngưỡng tải tối đa, phải có cơ chế shed load có kiểm soát (trả lỗi 503 kèm `Retry-After` cho một phần request) thay vì để tất cả request timeout đồng loạt.
- Đảm bảo thuật toán LB không route request "mua hàng" (ghi dữ liệu, cần nhất quán) và request "xem sản phẩm" (đọc, có thể cache) theo cùng một cách — phân tách được pool instance hoặc trọng số theo loại traffic.
- Test race condition: nhiều request cùng cập nhật bộ đếm connection/tải của một instance đồng thời — đảm bảo số liệu dùng để ra quyết định route không bị lệch do cập nhật không atomic.
- Có cơ chế circuit breaker phối hợp với LB: khi một instance liên tục trả lỗi 5xx, tự động loại khỏi vòng quay trong một khoảng thời gian rồi thử lại dần (half-open), không chờ health check thủ công.

---

## API gateway route theo tenant có trọng số cho nền tảng SaaS B2B

**Repository:** `load-balancing-saas-b2b-tenant-weighted-routing`

**Hệ thống:** Một API gateway đứng trước các service backend của nền tảng SaaS B2B, phục vụ nhiều tenant (công ty khách hàng) có quy mô rất khác nhau — một số tenant lớn gửi lượng request gấp hàng trăm lần tenant nhỏ.

**Vai trò của flow:** Load balancing ở đây không chỉ phân phối request đều tới các instance backend, mà còn phải đảm bảo công bằng tài nguyên giữa các tenant — một tenant lớn đột biến traffic không được phép chiếm hết capacity chung khiến các tenant nhỏ hơn bị chậm hoặc timeout.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế giới hạn tài nguyên theo tenant (ví dụ số connection đồng thời hoặc số request/giây tối đa mà một tenant được phép chiếm trên tổng capacity cụm) độc lập với thuật toán chọn instance, để một tenant có traffic bất thường bị giới hạn ở tầng gateway trước khi kịp làm nghẽn backend chung.
- Xử lý trường hợp một tenant lớn đang có traffic đột biến hợp lệ (không phải tấn công, ví dụ họ đang chạy chiến dịch của riêng họ) — phân biệt được với traffic bất thường/lỗi client để không chặn nhầm nhu cầu chính đáng, đồng thời vẫn bảo vệ được tenant khác.
- Đảm bảo việc đếm và giới hạn traffic theo tenant hoạt động đúng khi có nhiều instance gateway chạy song song (bộ đếm phải chia sẻ trạng thái giữa các instance gateway, không phải đếm riêng độc lập từng instance rồi mỗi instance tự cho qua một phần giới hạn dẫn đến tổng vượt ngưỡng thực tế).
- Cho phép cấu hình trọng số route khác nhau theo gói dịch vụ của tenant (tenant trả phí cao hơn được ưu tiên nhiều tài nguyên hơn khi cụm gần đạt tải tối đa) mà không cần phải tách riêng instance backend vật lý cho từng nhóm tenant, tránh lãng phí tài nguyên khi tenant lớn đang rảnh.
- Thiết kế test mô phỏng một tenant gửi traffic tăng đột biến 50 lần trong vài giây, đo độ trễ và tỷ lệ lỗi của các tenant khác trong cùng cụm — phải chứng minh được các tenant khác gần như không bị ảnh hưởng nhờ giới hạn tài nguyên theo tenant đã thiết kế.

---

## Load balancing tại edge server cho CDN video streaming

**Repository:** `load-balancing-cdn-video-streaming-edge`

**Hệ thống:** Một hệ thống CDN phục vụ video streaming, có nhiều edge node đặt gần người dùng theo khu vực địa lý, mỗi edge node chịu tải hàng nghìn viewer xem đồng thời.

**Vai trò của flow:** Load balancing ở tầng edge phải chọn được node vừa gần người dùng nhất (giảm độ trễ) vừa còn đủ tải, và xử lý đúng khi một edge node đang phục vụ nhiều viewer bỗng trở nên quá tải giữa lúc đang phát video cho họ.

**Yêu cầu cụ thể:**
- Việc chọn edge node cho một viewer mới phải kết hợp cả yếu tố địa lý (độ trễ mạng ước tính) lẫn tải hiện tại của node (số viewer đang xem, băng thông đang dùng) — không chỉ chọn node gần nhất theo khoảng cách địa lý đơn thuần nếu node đó đã gần đạt giới hạn băng thông.
- Xử lý trường hợp edge node đang phục vụ hàng nghìn viewer bỗng vượt ngưỡng tải (do một sự kiện đông người xem cùng lúc) — với các viewer mới, phải chuyển hướng sang node lân cận khác; với các viewer đang xem dở trên node đó, cần chiến lược giảm tải có kiểm soát (ví dụ hạ chất lượng bitrate một số phiên) thay vì để tất cả cùng bị giật/lag đồng loạt.
- Đảm bảo việc chuyển một viewer từ edge node quá tải sang node khác giữa lúc đang phát không làm gián đoạn luồng phát video mà người dùng nhận biết được — cần cơ chế chuyển tiếp êm (ví dụ player tự động reconnect tới node mới tại đúng thời điểm đang phát) tương tự cơ chế đổi kênh CDN, không phải ngắt và load lại từ đầu.
- Xử lý race condition khi số liệu tải của một edge node được cập nhật từ nhiều nguồn gần như đồng thời (viewer mới join, viewer rời đi, viewer đổi chất lượng bitrate) — quyết định route viewer mới phải dựa trên số liệu đủ mới, tránh dồn nhiều viewer mới vào cùng một node vì đọc phải số liệu tải đã lỗi thời một nhịp.
- Thiết kế cơ chế phát hiện một edge node bị suy giảm chất lượng dịch vụ dù vẫn "healthy" theo health check cơ bản (ví dụ tỷ lệ buffering/rebuffer của viewer trên node đó tăng cao dù node không báo lỗi hệ thống), và giảm dần trọng số route viewer mới tới node đó trước khi tình trạng trở nên nghiêm trọng hơn.
