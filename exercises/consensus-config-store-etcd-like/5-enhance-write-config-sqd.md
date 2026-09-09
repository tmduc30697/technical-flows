# Enhance sequence — Ghi config qua Raft (commit theo majority)

Đây là **enhance** của flow đã có ở base (`2-base-write-config-sqd.md`). So với base, ghi config nay phải qua leader của cluster 5 node, leader chỉ trả response cho client sau khi log entry đã replicate thành công tới majority (3/5) — đáp ứng yêu cầu 1 và yêu cầu 3 của đề bài, đồng thời expose metric replication latency p99.

```mermaid
sequenceDiagram
    actor Svc as Internal Service
    participant Leader as RAFT_NODE (leader, term=7)
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant F3 as Follower 3 (chậm/tạm mất kết nối)
    participant F4 as Follower 4 (đã down)

    Svc->>Leader: PUT config_entry(key, value)
    Leader->>Leader: Append RAFT_LOG_ENTRY(index=101, term=7, committed=false)
    par Replicate tới các follower
        Leader->>F1: AppendEntries(index=101, term=7)
        Leader->>F2: AppendEntries(index=101, term=7)
        Leader->>F3: AppendEntries(index=101, term=7)
        Leader->>F4: AppendEntries(index=101, term=7)
    end
    F1-->>Leader: ACK
    F2-->>Leader: ACK
    F3--xLeader: Timeout (mạng chập chờn)
    F4--xLeader: Không phản hồi (đã down)

    Leader->>Leader: Đếm ACK = leader + F1 + F2 = 3/5 → đạt majority
    Leader->>Leader: Đánh dấu RAFT_LOG_ENTRY(index=101, committed=true)
    Leader->>Leader: Apply vào CONFIG_ENTRY state machine
    Leader-->>Svc: 200 OK (chỉ trả sau khi committed)
    Leader->>Leader: Ghi CLUSTER_METRIC(replication_latency_p99_ms)

    Note over Leader,F4: Chỉ 2/5 node down (F4 và F3 timeout) nhưng vẫn đạt quorum 3/5, ghi vẫn thành công đúng yêu cầu chịu tối đa 2 node down
```
