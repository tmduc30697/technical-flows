# Base sequence — Xử lý nạp tiền on-chain (ngưỡng confirmation cố định, không theo dõi sau khi confirm)

Đây là **base**, flow "Xử lý nạp tiền" ở trạng thái hiện tại — hệ thống chờ một ngưỡng confirmation cố định chung cho mọi loại tài sản rồi cộng ngay vào ledger, không theo dõi tiếp giao dịch sau khi đã coi là "confirmed". Flow này liên quan mật thiết tới enhance vì đây chính là nguồn dữ liệu (ON_CHAIN_TRANSACTION, LEDGER_BALANCE) mà đối soát sẽ đọc và so khớp, và là nơi tồn tại lỗ hổng RBF/fork mà đề bài yêu cầu xử lý.

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant Watcher as Blockchain Watcher
    participant Tx as ON_CHAIN_TRANSACTION
    participant Ledger as LEDGER_BALANCE

    Customer->>Watcher: Gửi crypto tới địa chỉ ví nạp tiền được cấp
    Watcher->>Tx: Ghi ON_CHAIN_TRANSACTION(status=pending, block_height=null)

    loop Theo dõi block mới
        Watcher->>Tx: Cập nhật block_height khi giao dịch được đưa vào block
    end

    alt Đạt ngưỡng confirmation cố định (ví dụ 6 block, áp dụng chung mọi loại tài sản)
        Watcher->>Tx: Cập nhật status=confirmed
        Watcher->>Ledger: Cộng amount vào available_balance của khách hàng
        Ledger-->>Customer: Số dư khả dụng để giao dịch trên sàn
        Note over Watcher,Tx: Sau khi đánh dấu confirmed, hệ thống ngừng theo dõi giao dịch này, giả định nó đã final vĩnh viễn
    end
```
