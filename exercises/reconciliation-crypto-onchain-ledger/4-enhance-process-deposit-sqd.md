# Enhance sequence — Xử lý nạp tiền on-chain (ngưỡng theo tài sản, theo dõi RBF/fork sau khi confirm)

Đây là **enhance** của flow "Xử lý nạp tiền" đã có ở base. So với base, ngưỡng confirmation không còn cố định chung mà tra theo `CONFIRMATION_POLICY` của từng loại tài sản, và hệ thống tiếp tục theo dõi giao dịch ngay cả sau khi đã confirmed để phát hiện bị thay thế (RBF) hoặc đảo do fork, đảo ngược lại phần đã cộng ledger nếu cần (đáp ứng yêu cầu 2 và 4 của đề bài).

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant Watcher as Blockchain Watcher
    participant Policy as CONFIRMATION_POLICY
    participant Tx as ON_CHAIN_TRANSACTION
    participant Ledger as LEDGER_BALANCE

    Customer->>Watcher: Gửi crypto tới địa chỉ ví nạp tiền được cấp
    Watcher->>Tx: Ghi ON_CHAIN_TRANSACTION(status=pending, asset_type)
    Watcher->>Policy: Đọc required_confirmations theo asset_type

    loop Theo dõi block mới
        Watcher->>Tx: Cập nhật confirmations_count
    end

    alt confirmations_count đạt required_confirmations của đúng loại tài sản này
        Watcher->>Tx: Cập nhật status=confirmed
        Watcher->>Ledger: Cộng amount vào available_balance
        Ledger-->>Customer: Số dư khả dụng để giao dịch trên sàn

        Note over Watcher,Tx: Không dừng theo dõi ngay, vẫn tiếp tục quan sát thêm 1 khoảng thời gian đề phòng đảo chuỗi

        loop Tiếp tục theo dõi sau khi confirmed
            Watcher->>Tx: Kiểm tra tx_hash còn tồn tại trên nhánh chính của chuỗi
        end

        alt Giao dịch bị thay thế bởi phí gas cao hơn (RBF) hoặc bị đảo do fork
            Watcher->>Tx: Cập nhật is_replaced hoặc is_reorged=true, superseded_by_tx_hash nếu có
            Watcher->>Ledger: Đảo ngược phần đã cộng, đưa available_balance về trước khi ghi nhận giao dịch này
            Note over Ledger,Customer: Không để khách hàng tiếp tục dùng số dư từ 1 giao dịch thực chất không còn hiệu lực trên chuỗi
        end
    end
```
