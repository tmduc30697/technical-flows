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

---

## Giữ chỗ tồn kho cho sản phẩm flash sale

**Repository:** `distributed-lock-flash-sale-inventory-reservation`

**Hệ thống:** Sàn thương mại điện tử mở bán flash sale một sản phẩm số lượng giới hạn, hàng nghìn request mua hàng gửi tới cùng lúc trong vài giây đầu mở bán.

**Vai trò của flow:** Distributed lock theo từng sản phẩm (per-SKU) đảm bảo tại một thời điểm chỉ một tiến trình được đọc-và-trừ tồn kho cho sản phẩm đó, tránh oversell do nhiều request đọc cùng số tồn kho rồi cùng trừ.

**Yêu cầu cụ thể:**
- Lock phải được khóa ở granularity đúng mức (per-SKU, không lock toàn catalog gây nghẽn cổ chai, không lock quá lỏng theo warehouse chung gây oversell chéo sản phẩm), với TTL đủ ngắn để nhả nhanh cho request tiếp theo nhưng đủ dài để hoàn tất thao tác trừ kho — quá ngắn thì lock hết hạn giữa chừng gây 2 tiến trình cùng trừ, quá dài thì hàng nghìn request xếp hàng chờ gây timeout hàng loạt.
- Nếu tiến trình giữ lock crash ngay sau khi đã trừ số dư tồn kho trong DB nhưng chưa kịp tạo record đơn hàng/release lock, phải có cơ chế phát hiện trạng thái dở dang này khi lock hết hạn (ví dụ đối chiếu số đã trừ với đơn hàng tồn tại tương ứng) để hoàn lại tồn kho bị "kẹt" thay vì mất vĩnh viễn.
- Dùng fencing token gắn với lock để tránh trường hợp một tiến trình bị treo (GC pause, network delay) tưởng mình còn giữ lock, ghi trừ kho sau khi lock đã hết hạn và bị tiến trình khác giành mất — thao tác trừ kho với token cũ phải bị service tồn kho từ chối.
- Với hàng nghìn request cạnh tranh cùng 1 lock, phải có chiến lược fail-fast cho request không giành được lock trong thời gian ngắn (trả "hết hàng/thử lại" thay vì xếp hàng chờ vô hạn), tránh hàng đợi lock phình to làm nghẽn toàn hệ thống.
- Retry của client (do timeout network) không được gây trừ kho 2 lần cho cùng một yêu cầu mua — cần idempotency key gắn với request mua hàng độc lập với cơ chế lock, vì lock chỉ đảm bảo tuần tự xử lý chứ không đảm bảo request không bị gửi lặp.

---

## Phân việc cho worker pool xử lý hàng đợi công việc lớn

**Repository:** `distributed-lock-worker-pool-job-distribution`

**Hệ thống:** Nhiều worker instance chạy song song để xử lý hàng loạt job từ một hàng đợi công việc lớn (ví dụ xử lý file, gửi batch email), cần chia việc sao cho không trùng lặp và không bỏ sót.

**Vai trò của flow:** Distributed lock/leader election theo từng job (hoặc theo phân vùng job) đảm bảo mỗi job chỉ được đúng một worker nhận và xử lý tại một thời điểm, đồng thời không bỏ sót job khi worker chết giữa chừng.

**Yêu cầu cụ thể:**
- Worker phải "claim" một job bằng lock/lease có TTL trước khi bắt đầu xử lý; nếu worker chết giữa chừng (crash, bị kill do autoscale down) mà không release lock, TTL hết hạn phải tự đưa job đó về trạng thái "khả dụng" để worker khác nhận lại — không để job kẹt vĩnh viễn ở trạng thái "đang xử lý" bởi worker đã chết.
- Hai worker cùng lúc thấy job "chưa ai nhận" và cùng cố gắng claim (race condition khi liệt kê job và acquire lock không atomic với nhau) phải được giải quyết sao cho chỉ đúng một worker thắng, worker thua phải nhận biết ngay để không tiếp tục xử lý song song, tránh xử lý trùng và ghi kết quả hai lần.
- Nếu job cần thời gian xử lý lâu hơn TTL ước tính ban đầu (ví dụ file lớn bất thường), worker đang xử lý phải renew lease định kỳ; nếu renew thất bại (mất kết nối coordinator) trong khi job vẫn đang chạy thật, cần cơ chế idempotent hoặc kiểm tra trạng thái trước khi apply kết quả để chịu được việc worker khác nhận lại job đó khi lock hết hạn.
- Khi một worker bị autoscale terminate có báo trước (graceful shutdown signal), phải chủ động release lock/trả lại job đang giữ ngay lập tức thay vì chờ TTL hết hạn, để giảm độ trễ job bị "treo" chờ timeout tự nhiên.
- Cần cơ chế định kỳ quét các job ở trạng thái "đang xử lý" quá lâu so với TTL kỳ vọng (dead job detection) để phát hiện sớm rò rỉ lock hoặc lỗi hệ thống, tránh dựa hoàn toàn vào TTL tự nhiên mà không có giám sát chủ động.
