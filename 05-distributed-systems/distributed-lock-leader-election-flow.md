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
