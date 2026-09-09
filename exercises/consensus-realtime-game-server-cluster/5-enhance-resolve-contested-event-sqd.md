# Enhance sequence — Resolve contested event (log replicate qua majority)

Đây là **enhance**, cùng flow "resolve-contested-event" đã có ở base nhưng nay thay đổi hoàn toàn: thứ tự tranh chấp được xác định bằng thứ tự append vào `CONSENSUS_LOG` tại leader (không dùng `client_timestamp`), và log phải được replicate, xác nhận bởi majority mới commit. Đáp ứng yêu cầu 1 (thứ tự duy nhất dựa trên log replicate qua majority) và yêu cầu 2 (trade-off latency giữa chờ majority và áp dụng lạc quan) của đề bài.

```mermaid
sequenceDiagram
    actor ClientA as Client A
    actor ClientB as Client B
    participant Leader as Leader NODE (term hiện tại)
    participant Followers as Follower NODEs (consensus_group)
    participant State as MATCH_STATE

    ClientA->>Leader: INPUT_EVENT(hit, idempotency_key=kA)
    ClientB->>Leader: INPUT_EVENT(hit, idempotency_key=kB)
    Note over Leader: Leader nhận A trước B (theo thứ tự đến tại chính leader), append vào CONSENSUS_LOG theo đúng thứ tự này, không quan tâm client_timestamp

    alt Chế độ chờ majority xác nhận (an toàn, thêm latency)
        Leader->>Followers: Replicate CONSENSUS_LOG entry (term, index, event=A rồi event=B)
        Followers-->>Leader: ACK ghi log thành công
        Note over Leader: Đợi đủ ACK từ majority (bao gồm leader) mới commit
        Leader->>State: Commit và áp dụng theo đúng thứ tự A trước B
        Leader-->>ClientA: Broadcast kết quả thống nhất, "A thắng tranh chấp"
        Leader-->>ClientB: Broadcast kết quả thống nhất, "A thắng tranh chấp"
        Note over Leader,Followers: Thêm khoảng 1 vòng round-trip latency so với xử lý không đồng thuận, nhưng mọi client và node đều thấy cùng 1 kết quả
    else Chế độ áp dụng lạc quan (nhanh, có rủi ro rollback)
        Leader->>State: Áp dụng ngay theo thứ tự nhận tại leader, ghi TENTATIVE_STATE(confirmed=false)
        Leader-->>ClientA: Broadcast kết quả tạm thời, "A thắng tranh chấp" (chưa chắc chắn)
        Leader-->>ClientB: Broadcast kết quả tạm thời, "A thắng tranh chấp" (chưa chắc chắn)
        Leader->>Followers: Replicate CONSENSUS_LOG entry song song
        Followers-->>Leader: ACK majority xác nhận đúng thứ tự A trước B
        Leader->>State: Đánh dấu TENTATIVE_STATE(confirmed=true), giữ nguyên kết quả
        Note over Leader: Nếu majority xác nhận thứ tự khác với thứ tự đã áp dụng lạc quan, Leader phải gửi correction rollback cho client, đây là tần suất rollback cần đo lường
    end
```
