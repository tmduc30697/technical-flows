# Write-ahead log (WAL) & crash recovery/durability flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần đảm bảo dữ liệu không mất khi crash — storage engine tự xây, message queue broker, ledger ngân hàng số, order processing dùng cache in-memory, và node trong distributed KV store — nhằm luyện cách ghi log trước khi apply, replay đúng thứ tự, checkpoint, và đo RPO/RTO trong web app thực tế.

---

## Database engine tự xây dựng cho ứng dụng nội bộ

**Repository:** `wal-custom-database-engine`

**Hệ thống:** Một nhóm xây dựng storage engine đơn giản (kiểu key-value hoặc bảng) cho một ứng dụng nội bộ, cần đảm bảo dữ liệu không mất khi crash.

**Vai trò của flow:** WAL ghi mọi thay đổi ra log tuần tự trước khi áp dụng vào cấu trúc dữ liệu chính (in-memory hoặc B-tree trên đĩa), đảm bảo có thể phục hồi đúng trạng thái sau crash.

**Yêu cầu cụ thể:**
- Mọi write phải được fsync xuống đĩa vào WAL trước khi trả response "thành công" cho client (durability trước khi ack), không được ack sớm khi dữ liệu còn nằm trong buffer OS chưa chắc đã ghi đĩa.
- Sau khi crash (kill -9 giữa lúc ghi), khi khởi động lại phải replay WAL đúng thứ tự để tái tạo đúng trạng thái tại thời điểm crash, không thiếu không thừa transaction đã commit.
- Ghi log phải phát hiện được entry bị ghi dở (torn write, ví dụ crash giữa lúc ghi 1 record) và bỏ qua phần dở đó khi replay, không làm hỏng toàn bộ log.
- Có checkpoint định kỳ để giới hạn độ dài WAL cần replay (không phải replay từ đầu thời gian sử dụng), và checkpoint phải an toàn nếu crash xảy ra giữa lúc đang checkpoint.
- Đo lường: thời gian recovery (từ lúc start tới lúc sẵn sàng nhận request) tương ứng với kích thước WAL cụ thể, để biết giới hạn WAL tối đa cho phép theo SLA khởi động lại.

---

## WAL cho transaction log tài khoản ngân hàng số

**Repository:** `wal-digital-bank-transaction-log`

**Hệ thống:** App ngân hàng số ghi transaction chuyển tiền, đây là dữ liệu tuyệt đối không được mất hoặc ghi sai dù có crash phần cứng.

**Vai trò của flow:** Mọi thay đổi số dư phải được ghi WAL với đảm bảo durability cao nhất (có thể ghi đồng thời ra nhiều thiết bị/đĩa) trước khi coi giao dịch là hoàn tất.

**Yêu cầu cụ thể:**
- WAL phải được ghi ra ít nhất 2 vị trí lưu trữ vật lý độc lập (ví dụ 2 đĩa hoặc 2 node) trước khi trả kết quả "giao dịch thành công", chống mất dữ liệu khi 1 đĩa hỏng ngay lúc crash.
- Mỗi entry log phải có checksum để phát hiện log bị corrupt (do lỗi đĩa) và từ chối replay entry đó, thay vì âm thầm apply dữ liệu sai vào số dư tài khoản.
- Recovery sau crash phải là toàn-hoặc-không cho mỗi transaction (atomic) — không có trường hợp một giao dịch chuyển tiền chỉ trừ tiền người gửi mà chưa cộng tiền người nhận sau khi phục hồi.
- Phải log đủ thông tin để có thể replay ra một audit trail đầy đủ, phục vụ đối soát với ngân hàng đối tác/cơ quan quản lý sau này.
- Đo lường: RPO (recovery point objective, lượng dữ liệu tối đa có thể mất) phải bằng 0 cho transaction đã ack thành công, và RTO (thời gian phục hồi) phải được benchmark cụ thể theo kích thước log thực tế.

---

## Node lưu trữ trong distributed KV store dùng WAL cho từng node cục bộ

**Repository:** `wal-distributed-kv-store-node`

**Hệ thống:** Một node trong cụm KV store phân tán (đã có replication ở tầng cluster) vẫn cần WAL cục bộ để đảm bảo node đó tự phục hồi đúng sau crash, không chỉ dựa vào replica khác.

**Vai trò của flow:** WAL cục bộ đảm bảo dữ liệu ghi vào node không bị mất ngay cả khi chỉ một node duy nhất đó crash, giảm tải phải luôn rebuild từ replica (vốn tốn network/thời gian hơn).

**Yêu cầu cụ thể:**
- Node phải tự phục hồi từ WAL cục bộ trước khi thông báo với cluster rằng nó "sẵn sàng trở lại" (rejoin), tránh trường hợp trả dữ liệu cũ/thiếu trong lúc đang catch-up từ replica.
- Nếu WAL cục bộ bị corrupt hoàn toàn (đĩa hỏng), node phải tự phát hiện và chuyển sang chế độ rebuild-from-replica thay vì cố gắng replay dữ liệu hỏng, tránh đưa dữ liệu sai vào cluster.
- Có sự phối hợp rõ ràng giữa recovery cục bộ (WAL) và recovery ở tầng cluster (Raft log) — không được để 2 cơ chế mâu thuẫn nhau về thứ tự áp dụng thay đổi.
- Đo lường: so sánh thời gian một node phục hồi bằng WAL cục bộ (nhanh) versus phải rebuild toàn bộ từ replica qua network (chậm hơn nhiều), để quyết định ngưỡng khi nào nên dùng cách nào.
- Test kịch bản mất điện toàn bộ datacenter khiến nhiều node cùng crash đồng thời — đảm bảo mỗi node phục hồi độc lập đúng trạng thái của nó trước khi cluster đồng thuận lại state chung.
