# Enhance sequence — Leader election với randomized timeout

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base chỉ có 1 node, không cần bầu leader. Mỗi node dùng election timeout ngẫu nhiên 150-300ms để tránh nhiều node cùng trở thành candidate một lúc (split vote), và term tăng đơn điệu mỗi lần có cuộc bầu mới — đáp ứng yêu cầu 2 và yêu cầu 4 của đề bài, đồng thời expose metric leader election count.

```mermaid
sequenceDiagram
    participant N1 as Node 1 (leader cũ, term=6, vừa crash)
    participant N2 as Node 2
    participant N3 as Node 3
    participant N4 as Node 4
    participant N5 as Node 5

    Note over N1: Leader cũ (term=6) crash, không còn gửi heartbeat
    N2->>N2: Election timeout ngẫu nhiên 217ms hết hạn trước (không nhận heartbeat)
    N2->>N2: Chuyển sang candidate, tự tăng current_term = 7
    par Xin phiếu bầu
        N2->>N3: RequestVote(term=7, candidate=N2)
        N2->>N4: RequestVote(term=7, candidate=N2)
        N2->>N5: RequestVote(term=7, candidate=N2)
    end
    N3-->>N2: Vote granted (term=7 mới hơn term đã biết)
    N4-->>N2: Vote granted
    N5--xN2: Timeout (mạng chậm)

    N2->>N2: Nhận 3/5 phiếu (bản thân + N3 + N4) → đạt majority
    N2->>N2: Trở thành leader, role=leader, current_term=7
    N2->>N2: Ghi CLUSTER_METRIC(leader_election_count += 1)
    par Gửi heartbeat khẳng định vai trò leader
        N2->>N3: AppendEntries rỗng (heartbeat, term=7)
        N2->>N4: AppendEntries rỗng (heartbeat, term=7)
        N2->>N5: AppendEntries rỗng (heartbeat, term=7)
    end

    Note over N2,N5: Timeout ngẫu nhiên khiến N2 làm candidate trước, tránh N3/N4 cùng lúc trở thành candidate và chia phiếu (split vote)
```
