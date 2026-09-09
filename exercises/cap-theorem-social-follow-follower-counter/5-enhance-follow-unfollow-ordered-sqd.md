# Enhance sequence — Follow/unfollow (write quorum cao, version để xử lý race)

Đây là **enhance**, cùng flow "Follow/unfollow" đã có ở base nhưng nay thay đổi theo yêu cầu 2 và 3 của đề bài: ghi với write quorum cao hơn, và dùng version/timestamp để xác định đúng ý định cuối khi có 2 request follow/unfollow gần như đồng thời.

```mermaid
sequenceDiagram
    actor User
    participant App as Social App
    participant Policy as QUORUM_POLICY store
    participant Rel as FOLLOW_RELATIONSHIP store

    User->>App: Bấm Follow (t1), sau đó Unfollow gần như ngay lập tức (t2, double-tap)
    App->>Policy: Lấy quorum cho purpose=relationship_write (cao)

    App->>Rel: Ghi request Follow (last_action_timestamp=t1) tới node X
    App->>Rel: Ghi request Unfollow (last_action_timestamp=t2) tới node Y gần như đồng thời

    Rel->>Rel: Node Y nhận request Unfollow trước, so sánh t2 với version hiện tại → áp dụng, version tăng
    Rel->>Rel: Node X nhận request Follow sau đó nhưng mang timestamp t1 (cũ hơn t2 đã áp dụng)
    Rel->>Rel: So sánh t1 với last_action_timestamp đang lưu (t2) → t1 < t2 → bỏ qua, không ghi đè

    Rel-->>App: Trạng thái cuối cùng = kết quả của t2 (Unfollow) — đúng ý định cuối cùng của user
    App-->>User: Hiển thị đúng trạng thái "Following: false"
    Note over Rel: Không xảy ra trạng thái không xác định như base — nhờ so sánh timestamp trước khi ghi đè
```
