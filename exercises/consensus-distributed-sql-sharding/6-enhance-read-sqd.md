# Enhance sequence — Đọc linearizable qua read-index/heartbeat

Đây là **enhance** của flow đã có ở base (`3-base-read-sqd.md`). So với base (leader trả kết quả ngay không xác nhận vai trò), nay leader phải gửi heartbeat/read-index tới majority replica để xác nhận vẫn còn là leader hợp lệ trước khi trả kết quả đọc, tránh trả dữ liệu cũ khi đang bị network partition cô lập — đáp ứng đúng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Client
    participant Leader as Node bị nghi cô lập do partition (tưởng mình vẫn là leader Range B)
    participant F1 as Follower 1 (Range B)
    participant F2 as Follower 2 (Range B, đã theo leader mới ở phía cluster majority)

    Client->>Leader: Read row(key=42)
    Leader->>Leader: Ghi nhận read_index = commit_index hiện tại = 900
    par Xác nhận vẫn là leader hợp lệ (read-index confirmation)
        Leader->>F1: Heartbeat, xác nhận term hiện tại
        Leader->>F2: Heartbeat, xác nhận term hiện tại
    end
    F1--xLeader: Timeout (đang ở phía minority cùng Leader)
    F2--xLeader: Timeout (đã bị partition khỏi Leader)

    Leader->>Leader: Không đạt majority ACK cho heartbeat → không còn là leader hợp lệ
    Leader-->>Client: Reject, "not leader / cannot confirm read-index"

    Note over Leader,Client: Nếu heartbeat đạt majority ACK, leader mới được phép trả kết quả đọc tại read_index đó — đảm bảo tính linearizable
```
