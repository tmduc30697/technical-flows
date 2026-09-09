# Enhance sequence — Node failover (leader election + replay log, chống double-apply)

Đây là **enhance**, cùng flow "node-failover" đã có ở base nhưng nay thay đổi: thay vì tiếp quản dựa trên bản snapshot replicate async đã cũ, node mới được bầu qua leader election và phục hồi bằng cách replay `CONSENSUS_LOG` đã commit, đồng thời dùng `idempotency_key` để tránh mất hoặc áp dụng trùng input trong khoảng gap chuyển giao. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor Client
    participant Leader as Leader NODE (chết)
    participant Follower1 as Follower NODE 1
    participant Follower2 as Follower NODE 2
    participant State as MATCH_STATE

    Client->>Leader: INPUT_EVENT(idempotency_key=k1) ngay trước khi Leader chết
    Leader->>Follower1: Replicate CONSENSUS_LOG entry (term=5, index=42, event=k1)
    Note over Leader: Leader chết ngay sau khi gửi replicate cho Follower1, chưa kịp gửi Follower2 và chưa commit

    Note over Follower1,Follower2: Cả 2 phát hiện mất heartbeat từ Leader, bắt đầu bầu leader mới (term=6)
    Follower1->>Follower2: Request vote (term=6, log_index mới nhất=42)
    Follower2-->>Follower1: Vote cho Follower1 (log của Follower1 mới hơn hoặc bằng)
    Note over Follower1: Follower1 đạt majority vote, trở thành Leader mới term=6

    Follower1->>State: Replay CONSENSUS_LOG đã commit gần nhất để khôi phục MATCH_STATE
    Follower1->>Follower1: Kiểm tra entry index=42 (event k1) đã có trong log local nhưng chưa commit
    Follower1->>Follower1: Coi entry chưa commit là chưa chắc chắn, chờ input gốc hoặc client resend

    Client->>Follower1: Client timeout, gửi lại INPUT_EVENT(idempotency_key=k1)
    Follower1->>Follower1: Tra idempotency_key=k1, thấy đã tồn tại trong log (dù chưa commit) nên không tạo entry mới trùng
    Follower1->>State: Commit entry k1 duy nhất một lần, áp dụng vào state
    Follower1-->>Client: Xác nhận input đã được áp dụng, trận tiếp tục bình thường
    Note over Follower1: Không mất input (nhờ resend + log), không double-apply (nhờ idempotency_key), đây là thời gian phục hồi trung bình cần đo lường
```
