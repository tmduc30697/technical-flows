# Base sequence — Client retry sau timeout gây double-apply

Đây là **base**, mô tả 1 hệ quả trực tiếp của flow apply ở trên — khi client không nhận được response do timeout (dù giao dịch thực ra đã apply thành công), client tự retry và ledger apply lại lần nữa vì không có gì để nhận diện đây là cùng 1 giao dịch. Đây chính là vấn đề mà yêu cầu 4 (idempotency) trong đề bài muốn giải quyết.

```mermaid
sequenceDiagram
    actor Client
    participant Ledger as Ledger Node (duy nhất)

    Client->>Ledger: Chuyển 100$ từ account A sang account B
    Ledger->>Ledger: Trừ balance A, cộng balance B, ghi TRANSACTION
    Ledger--xClient: Response bị mất trên đường về (network timeout)

    Client->>Client: Không nhận được response, giả định request có thể đã thất bại
    Client->>Ledger: Retry chuyển 100$ từ account A sang account B (request giống hệt)
    Ledger->>Ledger: Không có cách nào biết đây là request đã xử lý trước đó
    Ledger->>Ledger: Trừ balance A, cộng balance B lần thứ 2, ghi thêm 1 TRANSACTION mới
    Ledger-->>Client: 200 OK

    Note over Ledger,Client: Account A bị trừ tổng cộng 200$ dù user chỉ có ý định chuyển 100$ một lần — double-apply
```
