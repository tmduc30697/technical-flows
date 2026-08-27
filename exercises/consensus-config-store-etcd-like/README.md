# Distributed config/feature-flag store nội bộ (etcd-like)

**Hệ thống:** Một cluster lưu config và feature flag dùng chung cho hàng trăm service nội bộ, cần đọc nhanh, ghi ít nhưng phải nhất quán.

**Vai trò của flow:** Dùng Raft để election leader chịu trách nhiệm nhận ghi và replicate log config change tới các follower, đảm bảo mọi node đồng thuận về giá trị config hiện tại.

**Yêu cầu cụ thể:**
- Cluster 5 node, chịu được tối đa 2 node down mà vẫn hoạt động (quorum = 3/5).
- Leader election dùng randomized election timeout (150-300ms) để tránh split vote liên tục.
- Log entry chỉ được coi là "committed" sau khi replicate thành công tới majority, và leader chỉ trả response cho client sau khi entry committed.
- Term number phải tăng đơn điệu; node nhận được request có term thấp hơn phải từ chối và tự chuyển về follower nếu đang là leader với term cũ.
- Có snapshot/log compaction định kỳ để log không phình vô hạn, và follower bị tụt quá xa phải nhận snapshot thay vì replay toàn bộ log.
- Đo lường: expose metric leader election count, log replication latency p99, và commit index lag giữa leader/follower.
