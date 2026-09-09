# Base sequence — Ghi dữ liệu (chỉ 1 range, 1 leader duy nhất)

Đây là **base**, flow "Client ghi dữ liệu" ở trạng thái hiện tại — vì toàn bộ bảng chỉ nằm trong 1 range với đúng 1 Raft group, client luôn biết chắc phải gửi tới đâu (không có nhiều range/nhiều leader để chọn nhầm). Flow này là tiền đề cho enhance vì khi bảng được chia thành nhiều range (yêu cầu 1, 2, 3), bài toán "gửi đúng leader" mới thực sự phát sinh.

```mermaid
sequenceDiagram
    actor Client
    participant Leader as RAFT_GROUP leader (duy nhất, phụ trách toàn bảng)
    participant F1 as Replica follower 1
    participant F2 as Replica follower 2

    Client->>Leader: Write row(key=42, value)
    Leader->>Leader: Append log entry
    par Replicate
        Leader->>F1: AppendEntries
        Leader->>F2: AppendEntries
    end
    F1-->>Leader: ACK
    F2-->>Leader: ACK
    Leader->>Leader: Majority đạt, commit
    Leader-->>Client: 200 OK

    Note over Client,Leader: Client không cần tra cứu "leader của range nào" vì cả cluster chỉ có 1 range, 1 leader
```
