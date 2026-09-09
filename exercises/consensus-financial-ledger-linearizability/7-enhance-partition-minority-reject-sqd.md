# Enhance sequence — Phía minority chuyển read-only khi network partition

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base chỉ có 1 node, không có khái niệm minority/majority. Khi network partition chia cluster, phía không đạt quorum phải tự chuyển sang read-only/reject-write, không tự tạo giao dịch mới, để tránh double-ledger khi partition được hàn lại — đáp ứng đúng yêu cầu 2 của đề bài, kèm cảnh báo khi mất quorum quá lâu.

```mermaid
sequenceDiagram
    actor Client
    participant Minority as LEDGER_NODE A, B (2 node, phía minority)
    participant Majority as LEDGER_NODE C, D, E (3 node, phía majority)

    Note over Minority,Majority: Network partition chia cluster 5 node thành 2-3

    Minority->>Minority: Chỉ liên lạc được 2/5 node, không đạt quorum 3/5
    Minority->>Minority: Tự chuyển quorum_side=minority, role=read-only

    Client->>Minority: Chuyển 100$ từ account A sang account B
    Minority-->>Client: Reject, "read-only mode, không đạt quorum, không tạo giao dịch mới"

    Client->>Minority: Đọc balance account A (chỉ đọc)
    Minority-->>Client: Trả balance hiện có kèm cảnh báo "có thể không phải giá trị mới nhất"

    loop Theo dõi thời gian mất quorum
        Minority->>Minority: seconds_without_quorum tăng dần
        alt seconds_without_quorum vượt ngưỡng X
            Minority->>Minority: Ghi QUORUM_ALERT(alert_triggered=true)
        end
    end

    Note over Minority,Majority: Phía majority vẫn tiếp tục nhận giao dịch mới bình thường vì đạt đủ quorum 3/5
```
