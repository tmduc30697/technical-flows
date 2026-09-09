# Enhance sequence — Node từ chối term cũ và tự chuyển về follower

Đây là **enhance**, flow hoàn toàn mới, đi sâu vào 1 khía cạnh riêng của yêu cầu 4 trong đề bài: term phải tăng đơn điệu, node nào nhận request có term thấp hơn phải từ chối, và node đang là leader với term cũ phải tự chuyển về follower ngay khi phát hiện term mới hơn — tránh tình trạng 2 "leader" cùng tồn tại (split-brain do term cũ).

```mermaid
sequenceDiagram
    participant OldLeader as Node 1 (leader cũ, term=6, vừa bị network partition, không biết đã có leader mới)
    participant NewLeader as Node 2 (leader mới, term=7)
    participant F3 as Node 3 (follower, đã biết term=7)

    Note over OldLeader: Trong lúc bị cô lập, Node 1 vẫn tin mình là leader với term=6
    OldLeader->>F3: AppendEntries(term=6, entries=[...])
    F3->>F3: So sánh term=6 (request) với current_term=7 (đã biết)
    F3-->>OldLeader: Reject, kèm current_term=7

    OldLeader->>OldLeader: Nhận current_term=7 > term=6 của bản thân
    OldLeader->>OldLeader: Tự cập nhật current_term=7, role: leader → follower
    Note over OldLeader: Không tự cho mình quyền tiếp tục làm leader chỉ vì chưa nhận heartbeat mới, dựa hẳn vào so sánh term

    NewLeader->>OldLeader: AppendEntries heartbeat (term=7)
    OldLeader-->>NewLeader: ACK (đã là follower, chấp nhận leader mới)

    Note over OldLeader,NewLeader: Term đơn điệu tăng là cơ chế duy nhất để cả cluster đồng thuận ai mới thực sự là leader hợp lệ
```
