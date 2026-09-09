# Enhance sequence — Leader mới xác định đúng offset đã commit trước khi cho publish tiếp

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base không có replica nên không có khái niệm "leader mới". Khi leader cũ chết đột ngột, broker mới lên thay phải so sánh log giữa các follower còn sống để xác định chính xác offset cuối cùng đã replicate đủ majority, tránh chọn nhầm offset dựa trên log chưa commit đầy đủ — đáp ứng đúng yêu cầu 2 của đề bài, đồng thời ghi nhận `failover_time_ms`.

```mermaid
sequenceDiagram
    participant OldLeader as Broker Leader cũ (term=4, vừa crash)
    participant F1 as Follower 1 (ứng viên leader mới, log tới offset=1005)
    participant F2 as Follower 2 (log tới offset=1000)

    Note over OldLeader: Leader cũ crash ngay sau khi gửi offset=1005 cho F1 nhưng chưa kịp gửi cho F2
    F1->>F1: Không còn nhận heartbeat, bắt đầu election, term=5
    F1->>F2: RequestVote(term=5, last_log_offset=1005)
    F2-->>F1: Vote granted (term=5 mới hơn, chấp nhận F1 làm leader)

    F1->>F1: Trở thành leader mới, term=5
    F1->>F2: So sánh log — F2 chỉ có tới offset=1000, chưa có offset=1001-1005
    Note over F1,F2: offset=1001-1005 chỉ tồn tại trên F1, KHÔNG đạt majority (chỉ 1/2 follower có), nên chưa từng được commit và chưa từng ack cho producer
    F1->>F1: Xác định last_committed_offset = 1000 (không phải 1005)
    F1->>F2: Replicate lại để đồng bộ tới offset=1000 (cắt bỏ 1001-1005 chưa commit)
    F2-->>F1: ACK, đồng bộ xong tới offset=1000

    F1->>F1: Cho phép nhận publish tiếp theo từ offset=1001 (đánh số lại)
    F1->>F1: Ghi FAILOVER_METRIC(failover_time_ms, leader_change_count += 1)

    Note over F1,F2: Vì offset 1001-1005 chưa từng ack cho producer nên việc "mất" chúng không vi phạm cam kết đã hứa, tránh trường hợp tệ hơn là giữ lại dữ liệu chưa commit gây phân nhánh
```
