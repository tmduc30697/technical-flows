# Consistent hashing & sharding/partitioning flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần chia dữ liệu ra nhiều node — resharding cụm cache, session store quy mô lớn, distributed rate limiter, time-series metrics, và lưu trữ tin nhắn theo conversation — nhằm luyện kỹ thuật consistent hashing/virtual node, xử lý rebalance khi scale, và tránh hot key/hot partition trong web app thực tế.

---

## Resharding cụm cache phân tán (Redis Cluster-like) khi scale

**Repository:** `consistent-hashing-redis-cluster-resharding`

**Hệ thống:** Một cụm cache dùng chung cho nhiều service, ban đầu có N node, cần scale thêm/giảm node theo tải mà không làm cache-miss toàn bộ.

**Vai trò của flow:** Consistent hashing quyết định key nào thuộc node nào, và khi thêm/bớt node chỉ một phần nhỏ key phải di chuyển (thay vì rehash toàn bộ).

**Yêu cầu cụ thể:**
- Khi thêm 1 node vào cụm N node, chỉ khoảng 1/(N+1) tổng số key phải di chuyển sang node mới — đo và chứng minh được tỉ lệ này bằng test thực tế.
- Dùng virtual node (nhiều điểm hash ảo cho mỗi node vật lý) để tránh phân bổ tải lệch (hot node) khi số node vật lý ít.
- Trong lúc resharding, request đọc/ghi vào key đang được di chuyển phải được xử lý đúng (không mất update, không đọc dữ liệu cũ) — nêu rõ chiến lược (dual-write tạm thời, hoặc route theo bảng ánh xạ đang chuyển tiếp).
- Có khả năng rollback resharding nếu phát hiện lỗi giữa quá trình migrate mà không làm mất dữ liệu đã migrate một phần.
- Đo lường: thời gian hoàn thành resharding cho một lượng dữ liệu cụ thể (ví dụ 100GB) và ảnh hưởng tới p99 latency của cache trong lúc resharding diễn ra.

---

## Sharding lưu trữ time-series metrics theo thời gian và theo nguồn

**Repository:** `consistent-hashing-time-series-metrics`

**Hệ thống:** Một hệ thống giám sát (monitoring) nhận metrics từ hàng chục nghìn service, cần lưu time-series data với throughput ghi rất cao.

**Vai trò của flow:** Partition dữ liệu theo cả thời gian (time-based partition) và theo nguồn (hash theo metric/service) để ghi/đọc song song hiệu quả, tránh một partition trở thành nghẽn cổ chai.

**Yêu cầu cụ thể:**
- Phải chọn chiến lược partition kết hợp (composite key: hash(service) + time bucket) và giải thích lý do tránh được "hot partition" khi một service đẩy metrics dồn dập.
- Truy vấn theo khoảng thời gian (ví dụ metrics 1 giờ qua của 1 service) phải chỉ chạm vào số partition tối thiểu cần thiết, không phải scan toàn bộ cụm.
- Dữ liệu partition cũ (quá X ngày) phải tự động được nén/archive sang storage rẻ hơn mà không làm gián đoạn ghi dữ liệu mới.
- Đảm bảo throughput ghi (write) tối thiểu (ví dụ 1 triệu điểm dữ liệu/giây toàn cụm) và nêu rõ số node/partition cần để đạt mức đó.
- Có cơ chế phát hiện partition bị lệch tải nghiêm trọng (một service quá lớn so với các service khác) và tách riêng partition dành cho nó (partition splitting theo nhu cầu).

---

## Sharding lưu trữ tin nhắn theo conversation cho app chat

**Repository:** `consistent-hashing-chat-message-sharding`

**Hệ thống:** App chat với hàng triệu cuộc hội thoại, mỗi cuộc hội thoại có thể có rất nhiều tin nhắn, cần chia dữ liệu ra nhiều shard DB.

**Vai trò của flow:** Consistent hashing/partition theo conversation ID để đảm bảo tất cả tin nhắn của một cuộc hội thoại nằm trên cùng shard (giữ tính cục bộ để đọc lịch sử nhanh, không cần join xuyên shard).

**Yêu cầu cụ thể:**
- Toàn bộ message của một conversation_id phải luôn thuộc đúng một shard xác định để truy vấn lịch sử chat không phải scatter-gather nhiều shard.
- Xử lý được "hot conversation" (group chat cực lớn, hàng trăm nghìn tin nhắn/ngày) không làm quá tải shard chứa nó — nêu chiến lược tách riêng (ví dụ shard riêng cho conversation vượt ngưỡng kích thước).
- Khi thêm shard mới, chỉ những conversation thuộc range bị dịch chuyển mới cần migrate, các conversation khác không bị ảnh hưởng gì (không downtime).
- Đảm bảo trong lúc migrate một conversation sang shard mới, tin nhắn gửi tới trong lúc migrate không bị mất hoặc ghi nhầm shard cũ.
- Đo lường: độ lệch kích thước dữ liệu giữa các shard theo thời gian, và cảnh báo khi một shard vượt X% dung lượng trung bình để chủ động rebalance trước khi đầy.

---

## Session store quy mô lớn cho hàng chục triệu user online

**Repository:** `consistent-hashing-largescale-session-store`

**Hệ thống:** Nền tảng lưu session của hàng chục triệu user đang online cùng lúc, phân tán session data ra nhiều node để chịu tải, cần đảm bảo tính sẵn sàng cao vì mất session đồng nghĩa user bị đăng xuất ngoài ý muốn.

**Vai trò của flow:** Consistent hashing quyết định session của một user thuộc node nào, và khi cụm phải scale (thêm/bớt node do thay đổi tải), việc rebalance session chỉ ảnh hưởng một phần nhỏ session, không được làm mất hàng loạt session cùng lúc khi node thay đổi.

**Yêu cầu cụ thể:**
- Khi thêm hoặc bớt node trong cụm lưu session, chỉ những session thuộc vùng hash bị dịch chuyển mới cần di chuyển sang node mới — dùng virtual node đủ nhiều để đảm bảo tải session không dồn lệch lên vài node vật lý ngay sau khi scale.
- Trong lúc đang rebalance (di chuyển session từ node cũ sang node mới), request của user có session đang được di chuyển phải vẫn được phục vụ đúng — cần chiến lược rõ ràng (double-read ở cả node cũ và mới trong giai đoạn chuyển tiếp, hoặc giữ bảng ánh xạ "đang di chuyển" để router biết hỏi node nào) để tránh user bị văng ra ngoài (session not found) giữa chừng dù session thực tế vẫn còn hợp lệ.
- Khi một node lưu session chết đột ngột (không phải scale có kế hoạch mà là sự cố), toàn bộ session trên node đó theo consistent hashing thuần sẽ dồn hết sang node kế tiếp trên ring — nếu không có replication trước đó, đây là mất session hàng loạt cho đúng dải user rơi vào node chết; cần nêu rõ chiến lược replicate session sang N node kế cận trên ring để chịu được việc mất một node mà không mất session.
- Session có TTL (hết hạn tự nhiên) chồng lấn với việc bị di chuyển do rebalance — cần đảm bảo TTL gốc được giữ nguyên khi session chuyển node (không bị reset TTL về đầy do quá trình di chuyển), tránh tình trạng session lẽ ra đã hết hạn tự nhiên nhưng lại "sống lại" đầy đủ TTL sau khi rebalance.
- Đo lường: tỉ lệ session bị lỗi "not found" tạm thời trong lúc scale cụm (session miss rate during rebalance), thời gian hoàn tất rebalance cho một đợt scale, và số lượng session thực sự bị mất (nếu có) sau sự cố node chết để đánh giá độ hiệu quả của chiến lược replication.

---

## Distributed rate limiter sharding theo user/API key

**Repository:** `consistent-hashing-distributed-rate-limiter`

**Hệ thống:** Bộ đếm rate limit (giới hạn số request cho phép trong một khoảng thời gian) theo từng user/API key, được sharding ra nhiều node để chịu tải lớn từ hàng triệu client gọi API cùng lúc.

**Vai trò của flow:** Consistent hashing đảm bảo mọi request của cùng một user/API key luôn được route tới đúng một node cố định để tính rate limit đúng (không bị đếm rải rác trên nhiều node dẫn tới giới hạn bị vô hiệu hóa), và khi cụm rate limiter scale/rebalance, việc ánh xạ key sang node phải ổn định để không làm counter bị reset hoặc nhân đôi limit ngoài ý muốn.

**Yêu cầu cụ thể:**
- Toàn bộ request của một user/API key phải luôn được route hash tới đúng một node chịu trách nhiệm đếm cho key đó — nếu router định tuyến sai (do bảng hash không đồng bộ giữa các instance router), rate limit thực tế của user đó có thể bị tính rải trên nhiều node khác nhau, khiến tổng số request cho phép vượt xa giới hạn thật (mỗi node lại tưởng user mới bắt đầu đếm từ 0).
- Khi cụm rate limiter thêm/bớt node (do scale theo tải), một số key sẽ bị ánh xạ sang node khác — counter hiện tại của các key đó (đang ở giữa cửa sổ tính giới hạn, ví dụ đang đếm 80/100 request trong phút hiện tại) phải được chuyển theo đúng giá trị sang node mới, không được để counter reset về 0 giữa chừng vì rebalance, vì như vậy vô tình cho phép user vượt giới hạn ngay sau khi cụm scale.
- Race condition khi nhiều request của cùng một key tới gần như đồng thời ngay tại thời điểm key đó đang được di chuyển giữa 2 node (do rebalance) — cần cơ chế rõ ràng tránh việc cả node cũ và node mới cùng đếm độc lập cho cùng một khoảng thời gian, dẫn tới user bị tính rate limit sai (chặn nhầm dù chưa vượt ngưỡng, hoặc ngược lại vượt ngưỡng thật nhưng không bị chặn).
- Với "hot key" (một API key/user có traffic cực lớn, ví dụ một service nội bộ gọi API liên tục với tần suất cao), việc dồn toàn bộ traffic tính rate limit cho key đó vào đúng một node theo consistent hashing thuần có thể biến node đó thành nghẽn cổ chai — cần nêu chiến lược xử lý (ví dụ chia nhỏ đếm cục bộ theo nhiều node rồi tổng hợp định kỳ/gần đúng cho riêng nhóm hot key, đánh đổi độ chính xác tức thời lấy khả năng chịu tải).
- Đo lường: độ trễ thêm vào do bước xác định node phụ trách rate limit cho mỗi request (routing overhead), tỉ lệ request bị tính sai giới hạn (đo bằng đối soát log request thực tế so với giới hạn cấu hình) trong lúc cụm đang rebalance, và tần suất phát hiện hot key cần xử lý riêng.
