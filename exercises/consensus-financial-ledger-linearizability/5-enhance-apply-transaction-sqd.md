# Enhance sequence — Apply giao dịch chỉ sau khi log entry commit

Đây là **enhance** của flow đã có ở base (`2-base-apply-transaction-sqd.md`). So với base (apply ngay khi nhận), nay giao dịch trước tiên là 1 `TRANSACTION_LOG_ENTRY`, chỉ được `committed=true` sau khi replicate tới majority node, và balance account chỉ cập nhật (apply state machine) sau bước commit này, không apply sớm — đáp ứng đúng yêu cầu 1 của đề bài, đồng thời ghi nhận metric commit latency.

```mermaid
sequenceDiagram
    actor Client
    participant Leader as LEDGER_NODE (leader, term=9)
    participant F1 as Follower 1
    participant F2 as Follower 2

    Client->>Leader: Chuyển 100$ từ account A sang account B (transaction_id=tx-789)
    Leader->>Leader: Append TRANSACTION_LOG_ENTRY(tx-789, committed=false)
    Note over Leader: Balance A/B CHƯA đổi ở bước này
    par Replicate
        Leader->>F1: AppendEntries(tx-789, term=9)
        Leader->>F2: AppendEntries(tx-789, term=9)
    end
    F1-->>Leader: ACK
    F2-->>Leader: ACK

    Leader->>Leader: Đạt majority (3/3) → đánh dấu committed=true
    Leader->>Leader: Apply state machine, trừ balance A, cộng balance B
    Leader-->>Client: 200 OK
    Leader->>Leader: Ghi LEDGER_METRIC(commit_latency_p50_ms, commit_latency_p99_ms)

    Note over Leader,F2: Nếu leader crash ngay trước khi đạt majority, TRANSACTION_LOG_ENTRY vẫn committed=false, balance chưa từng bị đổi — an toàn khi không có gì apply sớm
```
