# Distributed SQL database chia theo range/shard (CockroachDB-like)

**Hệ thống:** DB phân tán lưu dữ liệu giao dịch, chia dữ liệu thành nhiều "range" (shard), mỗi range có Raft group riêng gồm 3-5 replica.

**Vai trò của flow:** Mỗi range chạy Raft riêng để đồng thuận về thứ tự write trong range đó, đảm bảo linearizable read/write cho từng range dù node chết hay mạng chập chờn.

**Yêu cầu cụ thể:**
- Mỗi range phải có Raft group độc lập; leader của range A và range B có thể ở node khác nhau (multi-raft trên cùng cluster).
- Khi range bị split (do quá lớn) phải tạo Raft group mới cho phần tách ra, không làm mất write đang inflight.
- Client phải luôn được route tới leader hiện tại của range cần ghi; nếu gửi nhầm tới follower, follower trả redirect kèm địa chỉ leader mới nhất (nếu biết) hoặc lỗi NotLeader.
- Đảm bảo read từ leader là linearizable (leader phải xác nhận vẫn là leader hợp lệ bằng cách gửi heartbeat/read-index trước khi trả kết quả), tránh đọc dữ liệu cũ trong lúc network partition.
- Xử lý được kịch bản network partition chia cluster thành 2 nửa 2-3 hoặc 3-2: phía không đủ quorum không được nhận ghi.
