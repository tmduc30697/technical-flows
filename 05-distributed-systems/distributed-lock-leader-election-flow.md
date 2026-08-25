# Distributed lock & leader election flow (lease-based, Redis/ZooKeeper) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần khóa/bầu leader phân tán — cron job trên nhiều instance, giữ chỗ tồn kho flash sale, chống thundering herd cho cache, phân việc cho worker pool, và điều phối deploy đa vùng — nhằm luyện cơ chế lease-based lock, xử lý crash/fencing token và failover trong web app thực tế.

---

## Cron/scheduled job chạy trên nhiều instance service (horizontally scaled)

**Repository:** `distributed-lock-cron-job-scaled-instances`

**Hệ thống:** Một service backend được scale ra nhiều instance (pod) để chịu tải, nhưng có vài job định kỳ (gửi báo cáo, dọn dữ liệu) chỉ được chạy đúng 1 lần mỗi chu kỳ.

**Vai trò của flow:** Dùng distributed lock (lease-based, TTL) trên Redis/ZooKeeper để đảm bảo chỉ một instance giành được lock mới được chạy job, các instance khác skip.

**Yêu cầu cụ thể:**
- Lock phải có TTL (lease) rõ ràng; instance giữ lock phải renew (heartbeat) trước khi TTL hết, nếu không renew kịp thì lock tự nhả cho instance khác.
- Xử lý crash: nếu instance giữ lock crash giữa job (không kịp release lock), lock phải tự hết hạn sau TTL và job có thể chạy lại bởi instance khác — không bị "khoá chết" vĩnh viễn.
- Chống trường hợp instance cũ tưởng mình còn giữ lock (do GC pause/network delay) tiếp tục ghi dữ liệu sau khi lock đã hết hạn và bị instance khác giành mất (fencing token / lock version check trước khi ghi kết quả job).
- Log rõ instance nào giữ lock, thời điểm acquire/release/renew, để debug khi nghi ngờ job chạy trùng.
- Đo SLA: thời gian tối đa từ lúc lock cũ hết hạn tới lúc instance mới giành được lock mới (failover time) phải dưới một ngưỡng cụ thể (ví dụ 10s).

---

## Giữ chỗ tồn kho trong flash sale (inventory reservation)

**Repository:** `distributed-lock-flash-sale-inventory`

**Hệ thống:** Trang thương mại điện tử tổ chức flash sale, hàng nghìn user cùng bấm mua một sản phẩm limited stock trong vài giây.

**Vai trò của flow:** Dùng lock phân tán (per-SKU) để đảm bảo chỉ một request tại một thời điểm được trừ tồn kho cho sản phẩm đó, tránh oversell.

**Yêu cầu cụ thể:**
- Lock phải scope theo từng SKU (không lock toàn hệ thống) để không nghẽn cổ chai khi nhiều SKU khác nhau được mua song song.
- Thời gian giữ lock phải rất ngắn (chỉ đủ để check + trừ tồn kho, không giữ lock qua bước gọi payment gateway) để tối đa throughput.
- Nếu giữ lock quá X ms không xử lý xong phải tự nhả và trả lỗi timeout cho client, không để user chờ vô hạn.
- Phải benchmark throughput tối thiểu (ví dụ >= 500 request/giây/SKU) và đo được lock contention rate khi nhiều instance cùng tranh 1 SKU.
- Xử lý crash của Redis/coordinator node giữ lock: có failover sang node khác mà không làm mất tính đúng đắn của tồn kho (không cho oversell dù coordinator down).

---

## Leader coordinator cho cache invalidation tránh thundering herd

**Repository:** `distributed-lock-cache-thundering-herd`

**Hệ thống:** Hệ thống cache nhiều lớp cho một trang tin tức lớn, khi cache miss đồng loạt (ví dụ sau khi publish bài mới) hàng trăm request cùng dội xuống DB.

**Vai trò của flow:** Dùng leader election/distributed lock để chỉ một request/instance được phép đi warm-up cache/query DB, các request khác chờ hoặc dùng dữ liệu cũ.

**Yêu cầu cụ thể:**
- Chỉ request đầu tiên "giành" được lock cho một cache key mới được query DB và ghi lại cache; các request đến sau trong lúc lock còn giữ phải chờ ngắn (vài trăm ms) hoặc trả stale data nếu có, không tự query DB song song.
- Lock phải tự hết hạn nếu request giữ lock bị treo/crash, để tránh các request khác chờ vô hạn.
- Có cơ chế "single flight" nội bộ trong từng instance kết hợp với lock phân tán giữa các instance để giảm số round-trip tới coordinator.
- Đo lường: số lượng DB query giảm được so với baseline không có coordinator (mục tiêu giảm >90% query trùng lặp trong 1 giây đầu sau cache miss).
- Xử lý network partition giữa instance và Redis: instance phải có fallback (ví dụ tự query DB có giới hạn rate) khi không kết nối được coordinator, tránh outage toàn phần.

---

## Leader phân việc cho pool worker xử lý video

**Repository:** `distributed-lock-video-worker-pool-leader`

**Hệ thống:** Nền tảng chia sẻ video cần một cụm worker xử lý encode/transcode video upload, worker được scale động.

**Vai trò của flow:** Một worker được bầu làm leader (qua lease-based election) để phân chia job (assign video nào cho worker nào), các worker còn lại là follower chỉ xử lý job được giao.

**Yêu cầu cụ thể:**
- Leader phải định kỳ renew lease; nếu leader chết hoặc mất kết nối quá TTL, một follower khác phải tự động giành lease và trở thành leader mới trong thời gian giới hạn (ví dụ dưới 15s).
- Trong giai đoạn chuyển giao leader (không có leader), job mới không được assign nhưng job đang chạy trên worker khác vẫn phải tiếp tục, không bị hủy.
- Phải tránh split-brain: nếu leader cũ do network delay vẫn tưởng mình là leader và tiếp tục assign job sau khi lease hết hạn, hệ thống phải phát hiện và bỏ qua assignment đó (dựa vào fencing token/lease version).
- Có dashboard/metric hiển thị worker nào đang là leader, lịch sử chuyển leader, và số job bị assign trùng (nếu có) để giám sát.
- Test được kịch bản network partition chia worker thành 2 nhóm, đảm bảo tối đa một leader hợp lệ tồn tại tại một thời điểm.

---

## Điều phối deploy đa vùng (multi-region) tránh deploy trùng

**Repository:** `distributed-lock-multi-region-deploy-coordination`

**Hệ thống:** Một nền tảng SaaS chạy ở 3 region, có pipeline CI/CD tự động deploy, cần đảm bảo không có 2 region tự deploy đồng thời gây trạng thái không đồng nhất.

**Vai trò của flow:** Dùng distributed lock toàn cục (cross-region) để chỉ một pipeline deploy được chạy tại một thời điểm, các pipeline khác phải chờ tới lượt.

**Yêu cầu cụ thể:**
- Lock phải được đặt ở một coordinator có khả năng chịu lỗi cao (ví dụ ZooKeeper ensemble hoặc etcd cluster) không phụ thuộc vào bất kỳ region đang deploy.
- Nếu pipeline giữ lock bị treo (do lỗi CI) quá thời gian tối đa cho phép, lock phải tự nhả và có cảnh báo cho người vận hành, không chặn deploy khác vô thời hạn.
- Phải log đầy đủ audit: region nào deploy, lúc nào, ai trigger, để phục vụ rollback/điều tra sự cố.
- Xử lý kịch bản coordinator mất kết nối tạm thời với 1 region: region đó không được tự cho là "an toàn để deploy" nếu không xác nhận được lock hợp lệ.
- Đưa ra được giới hạn thời gian chờ tối đa cho pipeline đang chờ lock (queue timeout) và hành vi khi vượt quá (hủy job, thông báo, retry).
