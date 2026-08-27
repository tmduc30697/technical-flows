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

---

## Cụm game server real-time đồng thuận trạng thái trận đấu

**Repository:** `consensus-realtime-game-server-cluster`

**Hệ thống:** Cụm nhiều node game server xử lý các trận đấu real-time (multiplayer), cần đồng thuận về trạng thái trận đấu dùng chung khi có tranh chấp thời gian thực (ví dụ ai bắn trúng trước, ai chiếm điểm trước) và phải chịu được node chết giữa trận mà không làm hỏng trận đấu đang diễn ra.

**Vai trò của flow:** Consensus giúp các node đồng thuận về một trình tự sự kiện (event ordering) duy nhất cho trạng thái trận đấu dùng chung, đảm bảo mọi client thấy kết quả tranh chấp nhất quán, và khi node đang giữ trạng thái trận (authoritative node) chết giữa trận, một node khác phải tiếp quản mà không làm mất tiến trình hoặc gây kết quả sai lệch cho người chơi.

**Yêu cầu cụ thể:**
- Khi 2 sự kiện tranh chấp thời gian thực xảy ra gần như đồng thời (ví dụ 2 người chơi cùng bắn trúng 1 mục tiêu trong khoảng vài mili-giây), cụm phải có cơ chế xác định thứ tự duy nhất (dựa trên log được replicate qua majority, không dựa vào timestamp riêng của từng client vì client có độ trễ mạng khác nhau) để mọi node và mọi người chơi thấy cùng một kết quả tranh chấp, tránh tình trạng mỗi client "thấy" một người thắng khác nhau.
- Đây là hệ thống cực nhạy về độ trễ (mọi quyết định đồng thuận đều cộng thêm latency vào trải nghiệm chơi) — phải cân nhắc rõ trade-off giữa việc chờ đủ majority xác nhận trước khi áp dụng trạng thái (an toàn nhưng thêm độ trễ cảm nhận được) và áp dụng lạc quan trước rồi rollback nếu bị đảo ngược sau (nhanh nhưng có rủi ro hiển thị sai tạm thời cho người chơi).
- Khi node đang giữ vai trò authoritative cho một trận đấu chết đột ngột giữa trận, một node khác phải tiếp quản dựa trên log đã replicate gần nhất, nhưng cần xử lý rõ khoảng "gap" các sự kiện input từ người chơi gửi ngay trước/trong lúc chuyển giao — không được để mất hẳn input đó (người chơi cảm giác thao tác bị "nuốt") cũng không được áp dụng input đó hai lần trên node mới.
- Trong lúc một phần cụm bị network partition (một nhóm node mất kết nối với phần còn lại), các trận đấu đang chạy trên node ở phía minority phải được xử lý rõ ràng: hoặc chuyển trận sang node ở phía majority (chấp nhận gián đoạn ngắn, thông báo người chơi "đang kết nối lại") hoặc tạm dừng trận, tuyệt đối không để 2 phía cùng tiếp tục xử lý độc lập cùng một trận rồi tạo ra 2 kết quả khác nhau.
- Đo lường: latency thêm vào do bước đồng thuận cho các sự kiện tranh chấp (so với xử lý không đồng thuận), tần suất phải rollback trạng thái lạc quan do bị đảo ngược sau đồng thuận thật, và thời gian trung bình để một trận đấu phục hồi hoàn toàn sau khi node giữ trạng thái của nó bị chết.

---

## Message broker tự xây đồng thuận thứ tự message

**Repository:** `consensus-custom-message-broker-ordering`

**Hệ thống:** Một message broker tự xây (không dùng lại broker có sẵn) gồm nhiều broker node, phục vụ publish/subscribe cho nhiều consumer, cần đảm bảo consumer nhận message theo đúng thứ tự đã publish dù cụm broker có node chết hoặc leader broker bị thay đổi.

**Vai trò của flow:** Consensus giữa các broker node quyết định thứ tự chính thức (canonical order) của message trong một topic/partition, đảm bảo khi leader broker hiện tại chết và một broker khác lên thay, thứ tự message consumer nhận vẫn nhất quán và không bị mất/lặp message đã được xác nhận.

**Yêu cầu cụ thể:**
- Mọi message publish vào một topic/partition phải được gán thứ tự (offset) chỉ sau khi broker leader hiện tại đã replicate thành công tới majority broker node khác trong nhóm phụ trách partition đó — message chỉ được coi là "đã publish thành công" (ack cho producer) sau bước này, không phải ngay khi leader nhận được.
- Khi broker leader phụ trách một partition chết đột ngột, broker mới lên thay phải xác định chính xác offset cuối cùng đã thực sự commit (replicate đủ majority) trước khi cho phép publish tiếp — nếu chọn nhầm một offset dựa trên log chưa commit đầy đủ của leader cũ, có thể làm mất message đã ack cho producer hoặc gây phân nhánh thứ tự giữa các replica.
- Trong lúc leader mới đang được bầu (giai đoạn gián đoạn ngắn), phải quyết định rõ hành vi cho producer đang cố publish (từ chối rõ ràng để producer tự retry, hay buffer tạm ở phía client) và cho consumer đang đọc (tạm dừng nhận message mới ở đúng offset cuối cùng đã đọc, không được đọc nhảy cóc hoặc đọc trùng khi kết nối lại broker mới).
- Consumer group có thể đang đọc dở một partition đúng lúc leader chuyển giao — cần đảm bảo offset commit của consumer (điểm đã đọc tới đâu) được lưu độc lập với trạng thái broker leader, để khi consumer reconnect vào broker mới, nó tiếp tục đọc đúng từ vị trí đã dừng, không bị mất message (đọc thiếu) hoặc nhận lại message đã xử lý (đọc trùng) một cách không kiểm soát được.
- Đo lường: thời gian gián đoạn publish/consume trong mỗi lần chuyển leader (failover time), tần suất phải chuyển leader trong thực tế vận hành, và có test định kỳ mô phỏng chết leader giữa lúc đang publish tải cao để xác nhận không có message nào bị mất hoặc đảo thứ tự ngoài dự kiến.
