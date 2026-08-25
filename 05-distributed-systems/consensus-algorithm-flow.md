# Consensus algorithm flow (Raft/Paxos: leader election, log replication) — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần đồng thuận phân tán — config/feature-flag store, database phân tán chia range, cụm game server real-time, message broker tự xây, và sổ cái tài chính — nhằm luyện đủ các khía cạnh của leader election, log replication, và xử lý network partition/crash trong hệ thống web thực tế.

---

## Distributed config/feature-flag store nội bộ (etcd-like)

**Repository:** `consensus-config-store-etcd-like`

**Hệ thống:** Một cluster lưu config và feature flag dùng chung cho hàng trăm service nội bộ, cần đọc nhanh, ghi ít nhưng phải nhất quán.

**Vai trò của flow:** Dùng Raft để election leader chịu trách nhiệm nhận ghi và replicate log config change tới các follower, đảm bảo mọi node đồng thuận về giá trị config hiện tại.

**Yêu cầu cụ thể:**
- Cluster 5 node, chịu được tối đa 2 node down mà vẫn hoạt động (quorum = 3/5).
- Leader election dùng randomized election timeout (150-300ms) để tránh split vote liên tục.
- Log entry chỉ được coi là "committed" sau khi replicate thành công tới majority, và leader chỉ trả response cho client sau khi entry committed.
- Term number phải tăng đơn điệu; node nhận được request có term thấp hơn phải từ chối và tự chuyển về follower nếu đang là leader với term cũ.
- Có snapshot/log compaction định kỳ để log không phình vô hạn, và follower bị tụt quá xa phải nhận snapshot thay vì replay toàn bộ log.
- Đo lường: expose metric leader election count, log replication latency p99, và commit index lag giữa leader/follower.

---

## Distributed SQL database chia theo range/shard (CockroachDB-like)

**Repository:** `consensus-distributed-sql-sharding`

**Hệ thống:** DB phân tán lưu dữ liệu giao dịch, chia dữ liệu thành nhiều "range" (shard), mỗi range có Raft group riêng gồm 3-5 replica.

**Vai trò của flow:** Mỗi range chạy Raft riêng để đồng thuận về thứ tự write trong range đó, đảm bảo linearizable read/write cho từng range dù node chết hay mạng chập chờn.

**Yêu cầu cụ thể:**
- Mỗi range phải có Raft group độc lập; leader của range A và range B có thể ở node khác nhau (multi-raft trên cùng cluster).
- Khi range bị split (do quá lớn) phải tạo Raft group mới cho phần tách ra, không làm mất write đang inflight.
- Client phải luôn được route tới leader hiện tại của range cần ghi; nếu gửi nhầm tới follower, follower trả redirect kèm địa chỉ leader mới nhất (nếu biết) hoặc lỗi NotLeader.
- Đảm bảo read từ leader là linearizable (leader phải xác nhận vẫn là leader hợp lệ bằng cách gửi heartbeat/read-index trước khi trả kết quả), tránh đọc dữ liệu cũ trong lúc network partition.
- Xử lý được kịch bản network partition chia cluster thành 2 nửa 2-3 hoặc 3-2: phía không đủ quorum không được nhận ghi.

---

## Cụm game server đồng bộ trạng thái trận đấu thời gian thực

**Repository:** `consensus-game-server-match-state`

**Hệ thống:** Nền tảng game nhiều người chơi, mỗi trận được xử lý bởi một nhóm server để chịu lỗi (server chết giữa trận không làm mất trận).

**Vai trò của flow:** Dùng consensus để các server trong nhóm đồng thuận về thứ tự action của người chơi (input) và trạng thái trận đấu, đảm bảo mọi server có replica trạng thái giống nhau.

**Yêu cầu cụ thể:**
- Input của người chơi được coi là "log entry"; chỉ áp dụng vào state machine của trận sau khi được commit qua majority.
- Độ trễ commit phải nằm trong ngân sách real-time (ví dụ dưới 100ms ở điều kiện mạng LAN nội bộ) — nêu rõ trade-off giữa an toàn (chờ quorum) và độ trễ.
- Khi leader (server chủ trì trận) chết giữa trận, phải bầu leader mới từ replica còn log đầy đủ nhất, và trận tiếp tục mà người chơi không bị kick.
- Có cơ chế phát hiện follower rớt mạng tạm thời và bắt kịp (catch-up) log khi kết nối lại, không cần restart toàn trận.
- Log/snapshot phải được dọn sau khi trận kết thúc, không giữ vô hạn.

---

## Cụm broker message queue tự xây (Kafka-like) với leader theo partition

**Repository:** `consensus-message-queue-kafka-like`

**Hệ thống:** Message broker tự phát triển, dữ liệu chia theo partition, mỗi partition có nhiều replica để chịu lỗi.

**Vai trò của flow:** Consensus (kiểu Raft/ISR list) chọn leader cho từng partition, đảm bảo message được replicate tới đủ replica trước khi acknowledge producer.

**Yêu cầu cụ thể:**
- Producer có option chọn "acks" level: chỉ leader nhận (nhanh, rủi ro mất data khi leader chết trước khi replicate) hoặc chờ quorum ISR (an toàn hơn, chậm hơn) — implement cả hai và đo latency khác biệt.
- Khi leader của partition chết, phải bầu leader mới trong ISR (in-sync replica set), và replica không nằm trong ISR (tụt quá xa) không được lên leader để tránh mất data đã ack.
- Consumer đọc phải luôn đọc từ leader hiện tại (hoặc từ replica được đánh dấu đồng bộ đủ), không đọc dữ liệu chưa committed.
- Có cơ chế "unclean leader election" là tùy chọn tắt/mở rõ ràng — nêu rõ hệ quả mất data nếu bật.
- Log lại mọi lần chuyển leader (partition, leader cũ, leader mới, epoch/term) để phục vụ điều tra sự cố.

---

## Sổ cái giao dịch tài chính nội bộ (ledger) yêu cầu linearizability nghiêm ngặt

**Repository:** `consensus-financial-ledger-linearizability`

**Hệ thống:** Hệ thống ledger ghi nhận giao dịch tiền giữa các tài khoản trong một fintech, không cho phép mất hoặc double-apply giao dịch.

**Vai trò của flow:** Consensus đảm bảo mọi giao dịch được ghi theo đúng một thứ tự duy nhất và toàn bộ cluster đồng thuận, kể cả khi node chết hoặc mạng phân vùng.

**Yêu cầu cụ thể:**
- Mỗi giao dịch chỉ được commit khi đã replicate tới majority node, và số dư tài khoản chỉ được cập nhật sau khi log entry commit (state machine apply sau commit, không apply sớm).
- Trong trường hợp network partition, phần thiểu số (minority) phải chuyển sang chế độ read-only/reject-write, không tự ý tạo giao dịch mới để tránh double-ledger khi partition được hàn lại (healed).
- Phải có cơ chế phát hiện và log rõ "split-brain" giả định (hai leader cùng tồn tại do term cũ) và tự động vô hiệu hóa leader cũ ngay khi nó nhận ra term mới hơn.
- Idempotency: mỗi giao dịch có transaction ID duy nhất; nếu client retry do timeout, hệ thống không apply hai lần dù request đã thực sự thành công ở lần gửi trước.
- Đo lường/observability: expose được commit latency p50/p99, số lần leader election trong 24h, và alert khi cluster mất quorum quá X giây.
