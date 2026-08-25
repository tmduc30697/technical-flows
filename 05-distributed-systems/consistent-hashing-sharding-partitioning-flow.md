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

## Sharding session store cho hệ thống đăng nhập quy mô lớn

**Repository:** `consistent-hashing-session-store-sharding`

**Hệ thống:** Một platform có hàng chục triệu session đang active, cần chia session ra nhiều node lưu trữ để không một node nào quá tải.

**Vai trò của flow:** Consistent hashing theo session ID/user ID để xác định node lưu session, đảm bảo việc scale node không làm invalidate hàng loạt session đang hoạt động của user.

**Yêu cầu cụ thể:**
- Session của một user phải luôn map về đúng node xác định (deterministic) dựa trên session key, để bất kỳ instance API nào cũng tìm đúng node lưu session đó mà không cần broadcast.
- Khi một node lưu session chết, phải có replica/backup rõ ràng (ví dụ mỗi session ghi vào 2 node kế tiếp trên ring) để không làm user tự động bị logout khi node chết.
- Việc thêm/bớt node không được làm invalidate session của user không liên quan tới node bị thay đổi (chỉ ảnh hưởng đúng phần dữ liệu thuộc range bị dịch chuyển).
- Đo lường phân bổ tải giữa các node (độ lệch chuẩn số session/node) sau khi scale, đảm bảo không có node nào gánh quá X% tải trung bình.
- Có cơ chế giám sát và tự động rebalance nếu phát hiện một node bị lệch tải quá ngưỡng do virtual node phân bổ không đều.

---

## Sharding cho distributed rate limiter quy mô lớn

**Repository:** `consistent-hashing-rate-limiter-sharding`

**Hệ thống:** Một API gateway phục vụ hàng nghìn client, cần rate limit theo client/API key với counter được lưu phân tán trên nhiều node để chịu tải.

**Vai trò của flow:** Consistent hashing route request rate-limit-check của một client về đúng node giữ counter cho client đó, tránh cần đồng bộ toàn cục cho mỗi request.

**Yêu cầu cụ thể:**
- Mỗi client key phải map ổn định về đúng 1 (hoặc N) node chịu trách nhiệm counter cho key đó, giảm thiểu network hop khi check rate limit ở tần suất cao.
- Khi node giữ counter chết, phải định nghĩa rõ hành vi: fail-open (cho qua tạm, có thể vượt limit ngắn hạn) hay fail-closed (chặn tạm, an toàn hơn nhưng ảnh hưởng trải nghiệm) — và implement được cả logging cho quyết định này.
- Khi thêm node mới vào cụm rate limiter, client đang gần chạm limit không được bị "reset" limit về 0 một cách bất công (mất lịch sử counter khi bị chuyển node).
- Đo throughput tối đa hệ thống rate limiter xử lý được (request check/giây) và latency thêm vào mỗi request do bước hash + route.
- Test rebalance khi số node tăng gấp đôi, đảm bảo tải phân bổ đều và không có node nào bị nghẽn ("hot key" của một client có traffic rất lớn).

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
