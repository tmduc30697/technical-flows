# Enhance sequence — Đối soát định kỳ ví on-chain vs ledger nội bộ

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có bước so khớp tổng tài sản). Đáp ứng yêu cầu 1 và 2 của đề bài: tính đúng số dư on-chain tại 1 block height xác định, so khớp với snapshot ledger cùng thời điểm, chỉ tính giao dịch đã confirmed (loại trừ pending).

```mermaid
sequenceDiagram
    participant Scheduler as Reconciliation Scheduler
    participant Chain as Blockchain (các ví hot/cold)
    participant Tx as ON_CHAIN_TRANSACTION
    participant Ledger as LEDGER_BALANCE
    participant Run as RECONCILIATION_RUN
    participant Discrepancy as RECONCILIATION_DISCREPANCY

    Scheduler->>Chain: Chốt block_height xác định làm mốc đối soát (ví dụ block mới nhất đã đủ độ sâu an toàn)
    Scheduler->>Chain: Tính tổng số dư thực tế trên toàn bộ ví hot/cold tại đúng block_height đó

    Scheduler->>Tx: Lấy toàn bộ ON_CHAIN_TRANSACTION có status=confirmed tính tới block_height, loại trừ pending/replaced/reorged
    Scheduler->>Ledger: Lấy snapshot tổng available_balance của toàn bộ khách hàng tại cùng thời điểm tương ứng (block_time)

    Note over Scheduler,Ledger: Snapshot ledger phải khớp đúng thời điểm với block_height, tránh so sánh lệch thời điểm gây sai lệch giả

    Scheduler->>Run: Tạo RECONCILIATION_RUN(block_height, block_time, onchain_total_balance, ledger_total_balance)
    Scheduler->>Scheduler: Tính discrepancy_amount = ledger_total_balance - onchain_total_balance

    alt discrepancy_amount xấp xỉ 0 trong sai số cho phép
        Scheduler->>Run: Cập nhật status=matched
    else Có sai lệch vượt sai số cho phép
        Scheduler->>Run: Cập nhật status=discrepancy_detected
        Scheduler->>Discrepancy: Ghi RECONCILIATION_DISCREPANCY(asset_type, discrepancy_type, amount, severity)
    end

    Note over Run,Discrepancy: Toàn bộ RECONCILIATION_RUN và RECONCILIATION_DISCREPANCY được lưu trữ minh bạch, không ghi đè, phục vụ audit về sau
```
