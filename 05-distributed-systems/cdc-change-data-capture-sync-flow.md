# CDC (Change Data Capture) sync flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần đồng bộ dữ liệu gần thời gian thực từ nguồn OLTP — data warehouse phân tích, search index e-commerce, cache warm-up, read replica đa vùng, và migrate legacy theo strangler fig — nhằm luyện cách đọc transaction log an toàn, giữ thứ tự thay đổi, và phục hồi đúng offset sau gián đoạn trong web app thực tế.

---

## Đồng bộ dữ liệu OLTP sang data warehouse cho phân tích

**Repository:** `cdc-oltp-to-data-warehouse`

**Hệ thống:** Một hệ thống bán hàng có DB giao dịch (OLTP) chính, cần đồng bộ dữ liệu gần thời gian thực sang data warehouse để BI/report.

**Vai trò của flow:** CDC đọc binlog/WAL của DB nguồn để bắt mọi thay đổi (insert/update/delete) và đẩy sang warehouse mà không cần query trực tiếp DB nguồn (tránh tải).

**Yêu cầu cụ thể:**
- Không được polling bằng query định kỳ lên bảng nguồn; phải đọc trực tiếp transaction log (binlog MySQL/WAL Postgres) để không tạo tải phụ lên DB production.
- Xử lý được schema change ở nguồn (thêm/xóa cột) mà không làm crash pipeline hoặc làm sai lệch dữ liệu đã đồng bộ trước đó.
- Đảm bảo thứ tự thay đổi trong cùng một bảng được giữ nguyên khi đồng bộ (không để update xảy ra trước insert do xử lý song song sai thứ tự).
- Có cơ chế resume từ đúng vị trí (log sequence number/offset) khi CDC connector bị restart, không đọc lại từ đầu hoặc bỏ sót thay đổi.
- Đo lường replication lag (khoảng cách thời gian giữa thay đổi ở nguồn và khi dữ liệu xuất hiện ở warehouse) và alert khi lag vượt ngưỡng (ví dụ >5 phút).

---

## Đồng bộ catalog sản phẩm vào search index (Elasticsearch)

**Repository:** `cdc-product-catalog-elasticsearch-sync`

**Hệ thống:** Trang e-commerce lưu catalog sản phẩm ở DB chính, nhưng search/filter chạy trên Elasticsearch cần đồng bộ theo thời gian thực khi sản phẩm được thêm/sửa/xóa.

**Vai trò của flow:** CDC bắt thay đổi ở bảng sản phẩm và đẩy vào pipeline index lại Elasticsearch, giữ search results luôn phản ánh đúng dữ liệu mới nhất.

**Yêu cầu cụ thể:**
- Update giá/tồn kho sản phẩm phải phản ánh lên search index trong vòng vài giây, không được có độ trễ lớn gây hiển thị sai giá cho khách.
- Xóa sản phẩm (soft delete hoặc hard delete) ở DB nguồn phải dẫn tới xóa/ẩn đúng document tương ứng trên Elasticsearch, không để sót sản phẩm đã xóa vẫn hiện trong search.
- Xử lý được trường hợp một sản phẩm bị sửa nhiều lần liên tiếp trong thời gian ngắn — index cuối cùng phải phản ánh trạng thái mới nhất, không bị ghi đè bởi event cũ tới muộn (out-of-order delivery).
- Khi cần rebuild toàn bộ index (ví dụ đổi mapping), phải có cách bootstrap lại từ full snapshot rồi mới tiếp tục CDC stream từ đúng điểm bắt đầu snapshot, không có khoảng trống dữ liệu.
- Giám sát: đếm được số document lệch giữa DB nguồn và index (consistency check định kỳ) để phát hiện sự cố đồng bộ ngầm.

---

## Cache warm-up/sync tự động khi dữ liệu gốc thay đổi

**Repository:** `cdc-cache-warmup-autosync`

**Hệ thống:** Một API service dùng cache (Redis) để tăng tốc đọc dữ liệu người dùng/sản phẩm, dữ liệu gốc nằm ở DB quan hệ.

**Vai trò của flow:** CDC lắng nghe thay đổi ở DB để tự động invalidate/update cache liên quan, thay cho cách cache theo TTL cố định dễ bị stale.

**Yêu cầu cụ thể:**
- Mỗi thay đổi ở bảng nguồn phải map được sang đúng (các) cache key cần invalidate — định nghĩa rõ mapping bảng/điều kiện → cache key pattern.
- Độ trễ từ lúc DB thay đổi tới lúc cache được invalidate/update phải đo được và có SLA cụ thể (ví dụ dưới 2 giây ở 99% trường hợp).
- Xử lý race condition: một request đọc cache đúng lúc CDC đang invalidate — không được để cache ở trạng thái nửa cũ nửa mới (ví dụ dùng versioning hoặc lock ngắn khi update cache).
- Nếu CDC pipeline bị down một khoảng thời gian, khi phục hồi phải phát hiện và invalidate toàn bộ cache có khả năng bị stale trong khoảng downtime đó, không giả định "chắc vẫn đúng".
- Có công cụ kiểm tra định kỳ (sampling) so sánh giá trị cache với DB gốc để phát hiện sai lệch còn sót.

---

## Đồng bộ read replica cho phân tích ở khu vực khác (cross-region analytics)

**Repository:** `cdc-cross-region-read-replica`

**Hệ thống:** Hệ thống chính chạy ở một region, cần một bản sao dữ liệu ở region khác chỉ dùng cho phân tích/báo cáo, không ảnh hưởng tới hiệu năng ghi ở region chính.

**Vai trò của flow:** CDC stream thay đổi từ DB chính xuyên region tới hệ thống phân tích, tách biệt hoàn toàn khỏi đường ghi giao dịch chính.

**Yêu cầu cụ thể:**
- Việc đọc CDC log từ DB chính tuyệt đối không được làm chậm transaction ghi ở region chính (đọc log ở mức OS/replication thread riêng, không lock table).
- Phải nêu rõ mức độ "eventual consistency" cụ thể — ví dụ dữ liệu phân tích có thể trễ tới bao nhiêu giây/phút so với dữ liệu gốc, và điều này được thông báo rõ cho người dùng báo cáo.
- Xử lý gián đoạn mạng liên vùng kéo dài (vài giờ): pipeline phải tự buffer hoặc dừng an toàn và catch-up đầy đủ khi mạng khôi phục, không mất thay đổi nào.
- Dữ liệu nhạy cảm (PII) phải được lọc/mask ngay trong pipeline CDC trước khi tới hệ thống phân tích ở region khác, tuân thủ yêu cầu compliance dữ liệu xuyên biên giới.
- Có metric đo replication lag theo thời gian thực và dashboard cảnh báo khi lag vượt SLA đã cam kết.

---

## Chiến lược "strangler fig" migrate dần từ hệ thống cũ sang hệ thống mới

**Repository:** `cdc-strangler-fig-migration`

**Hệ thống:** Một hệ thống legacy đang được thay dần bằng hệ thống mới; trong giai đoạn chuyển tiếp cả hai hệ thống cùng tồn tại và cần dữ liệu đồng bộ hai chiều.

**Vai trò của flow:** CDC đọc thay đổi từ hệ thống cũ để đẩy sang hệ thống mới (và ngược lại nếu cần), giữ hai hệ thống nhất quán trong suốt quá trình migrate từng phần chức năng.

**Yêu cầu cụ thể:**
- Xác định rõ "nguồn sự thật" (source of truth) cho từng loại dữ liệu ở từng giai đoạn migrate, tránh vòng lặp đồng bộ vô hạn (A đổi → sync sang B → B lại sync ngược về A).
- Có cơ chế đánh dấu nguồn gốc thay đổi (metadata "changed-by-sync" hay "changed-by-user") để tránh CDC tự bắt lại thay đổi do chính nó tạo ra và tạo loop.
- Xử lý xung đột khi cùng một record bị sửa gần như đồng thời ở cả 2 hệ thống trong giai đoạn transition — định nghĩa rule ưu tiên rõ ràng (last-write-wins theo timestamp, hoặc ưu tiên hệ thống mới).
- Có khả năng tắt/bật đồng bộ theo từng bảng/entity riêng lẻ để migrate dần từng phần mà không phải big-bang cutover toàn bộ.
- Log đầy đủ mọi bản ghi được đồng bộ (nguồn, đích, thời điểm, kết quả) để hỗ trợ điều tra khi có sai lệch dữ liệu bị phát hiện sau này.
