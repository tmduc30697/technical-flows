# Base sequence — Apply giao dịch ngay khi node nhận được

Đây là **base**, flow "Ghi nhận 1 giao dịch chuyển tiền" ở trạng thái hiện tại — node duy nhất cập nhật `balance` ngay khi nhận request, không có bước replicate hay chờ commit nào. Flow này là tiền đề cho enhance vì yêu cầu 1 của đề bài chính là tách rõ 2 bước "commit" và "apply", vốn ở base đang gộp làm một.

```mermaid
sequenceDiagram
    actor Client
    participant Ledger as Ledger Node (duy nhất)

    Client->>Ledger: Chuyển 100$ từ account A sang account B
    Ledger->>Ledger: Trừ balance A, cộng balance B ngay lập tức
    Ledger->>Ledger: Ghi TRANSACTION(applied_at=now)
    Ledger-->>Client: 200 OK

    Note over Ledger: Không có node dự phòng nào xác nhận giao dịch này, và balance đã đổi ngay cả khi chưa có gì đảm bảo giao dịch "chắc chắn" đã được ghi nhận bền vững
```
