# Enhance sequence — Phát hiện split-brain giả định và tự vô hiệu hoá leader cũ

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base không có nhiều node nên không thể có 2 leader cùng lúc. Khi partition hàn lại và 1 node vẫn tưởng mình là leader với term cũ trong khi cluster đã có leader mới với term cao hơn, hệ thống phải phát hiện, log rõ đây là split-brain giả định, và tự động vô hiệu hoá leader cũ ngay khi nó biết term mới — đáp ứng đúng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    participant OldLeader as LEDGER_NODE A (leader cũ, term=9, vừa hết bị cô lập)
    participant NewLeader as LEDGER_NODE C (leader mới, term=11, đã lên thay trong lúc A bị cô lập)
    participant F as LEDGER_NODE D (follower, đã biết term=11)

    Note over OldLeader: Partition vừa được hàn lại, A chưa kịp biết cluster đã có leader mới
    OldLeader->>F: AppendEntries(term=9, ...)
    F->>F: So sánh term=9 (request) với current_term=11 (đã biết)
    F-->>OldLeader: Reject, kèm current_term=11

    OldLeader->>OldLeader: Phát hiện current_term=11 > 9 của bản thân
    OldLeader->>OldLeader: Ghi SPLIT_BRAIN_LOG(old_leader=A, old_term=9, new_term_discovered=11)
    Note over OldLeader: Log rõ đây là split-brain giả định — 2 node cùng tin mình là leader tại 1 thời điểm do lệch term
    OldLeader->>OldLeader: Tự cập nhật current_term=11, role: leader → disabled_old_leader
    OldLeader->>OldLeader: Ghi old_leader_disabled_at = now vào SPLIT_BRAIN_LOG

    NewLeader->>OldLeader: AppendEntries heartbeat (term=11)
    OldLeader-->>NewLeader: ACK (đã tự vô hiệu hoá vai trò leader cũ, chấp nhận leader mới)

    Note over OldLeader,NewLeader: Term cao hơn luôn thắng, không cần trọng tài bên ngoài để quyết định ai mới là leader hợp lệ
```
