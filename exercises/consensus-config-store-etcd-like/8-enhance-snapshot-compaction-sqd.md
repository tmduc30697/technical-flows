# Enhance sequence — Snapshot/log compaction và follower tụt xa nhận snapshot

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base không có log để compact. Mỗi node định kỳ tự tạo `SNAPSHOT` để log không phình vô hạn, và follower bị tụt quá xa (log leader đã compact phần follower cần) sẽ nhận snapshot thay vì replay lại toàn bộ log — đáp ứng yêu cầu 5 của đề bài, đồng thời minh hoạ metric commit index lag.

```mermaid
sequenceDiagram
    participant Leader as RAFT_NODE (leader, commit_index=5000)
    participant F1 as Follower 1 (commit_index=4990, bám sát)
    participant F5 as Follower 5 (commit_index=800, mất kết nối lâu, tụt xa)

    loop Định kỳ mỗi N entry hoặc mỗi khoảng thời gian
        Leader->>Leader: Tạo SNAPSHOT(last_included_index=5000)
        Leader->>Leader: Xoá RAFT_LOG_ENTRY có index <= 5000 - retain_window
    end

    Leader->>F1: AppendEntries(from index=4991)
    F1-->>Leader: ACK, commit_index=5000

    F5->>Leader: Reconnect sau thời gian dài mất kết nối, xin entries từ index=801
    Leader->>Leader: Kiểm tra index=801 đã bị compact khỏi log (log chỉ còn từ index=4500)
    Leader-->>F5: InstallSnapshot(last_included_index=5000, state_snapshot)
    F5->>F5: Nạp snapshot, thay thế toàn bộ state hiện tại, commit_index=5000
    F5-->>Leader: ACK InstallSnapshot
    Leader->>F5: AppendEntries(from index=5001) tiếp tục bình thường

    Leader->>Leader: Ghi CLUSTER_METRIC(commit_index_lag = leader.commit_index - follower.commit_index)
    Note over Leader,F5: Nếu không có snapshot, Follower 5 sẽ phải replay lại hơn 4000 log entry đã bị xoá, không còn khả thi
```
