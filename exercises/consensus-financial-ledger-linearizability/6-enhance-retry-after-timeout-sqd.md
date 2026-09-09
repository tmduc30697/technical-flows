# Enhance sequence — Retry sau timeout không double-apply (idempotency)

Đây là **enhance** của flow đã có ở base (`3-base-retry-after-timeout-sqd.md`). So với base (apply lại vô điều kiện gây double-apply), nay mỗi giao dịch mang `transaction_id` duy nhất do client sinh ra — trước khi apply, leader kiểm tra `transaction_id` đã tồn tại trong `TRANSACTION_LOG_ENTRY` chưa, nếu có rồi thì trả lại đúng kết quả cũ thay vì apply lần 2 — đáp ứng đúng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Client
    participant Leader as LEDGER_NODE (leader)

    Client->>Leader: Chuyển 100$ từ A sang B (transaction_id=tx-789)
    Leader->>Leader: Kiểm tra tx-789 chưa tồn tại → tiến hành append + replicate + commit như bình thường
    Leader->>Leader: Apply, trừ A, cộng B
    Leader--xClient: Response bị mất trên đường về (network timeout)

    Client->>Client: Không nhận được response, tự retry với transaction_id GIỮ NGUYÊN = tx-789
    Client->>Leader: Retry chuyển 100$ từ A sang B (transaction_id=tx-789)
    Leader->>Leader: Kiểm tra transaction_id=tx-789 đã tồn tại trong TRANSACTION_LOG_ENTRY và committed=true
    Leader->>Leader: KHÔNG apply lại, chỉ lấy lại kết quả đã có
    Leader-->>Client: 200 OK (kết quả của lần commit trước đó, không trừ tiền thêm lần nữa)

    Note over Leader,Client: Account A chỉ bị trừ đúng 100$ một lần dù client gửi request 2 lần, vì idempotency dựa trên transaction_id chứ không dựa vào việc client có nhận được response hay không
```
