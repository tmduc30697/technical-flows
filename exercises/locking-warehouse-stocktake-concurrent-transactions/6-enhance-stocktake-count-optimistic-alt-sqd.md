# Enhance sequence — Stocktake count (phương án thay thế: optimistic, timestamp + đối chiếu log)

Đây là **enhance**, phương án thay thế cho flow `stocktake-count` khi tổ chức chấp nhận đánh đổi độ chính xác tuyệt đối để không làm gián đoạn vận hành kho. Thay vì khóa cứng khu vực (xem file `5-enhance-stocktake-count-sqd.md`), hệ thống chỉ ghi nhận `snapshot_started_at`, để mọi giao dịch xuất/nhập vẫn chạy bình thường trong lúc đếm, rồi khi submit sẽ đối chiếu với log `STOCK_TRANSACTION` xảy ra trong khoảng thời gian đếm để tính lại số kỳ vọng đúng. Đáp ứng phần còn lại của yêu cầu 1 (phương án optimistic được đề bài yêu cầu đánh giá song song với khóa cứng).

```mermaid
sequenceDiagram
    actor NV as Nhân viên kiểm kho
    participant App as Stocktake App
    participant DB as Database
    participant OMS as Hệ thống xử lý đơn hàng

    NV->>App: Bắt đầu kiểm kho khu vực A lúc 9:00 (strategy=optimistic_reconcile)
    App->>DB: INSERT STOCKTAKE (area=A, snapshot_started_at=9:00), không khóa area A
    DB-->>App: OK

    NV->>NV: Đếm thủ công SKU X tại khu vực A, đếm được 50

    par Song song trong lúc đang đếm, không bị chặn
        OMS->>DB: Phiếu xuất kho 5 đơn vị SKU X cho khu vực A
        DB->>DB: INSERT STOCK_TRANSACTION (area=A, sku=X, type=outbound, qty_delta=-5, created_at=9:10)
        DB->>DB: UPDATE INVENTORY_ITEM SET quantity = quantity - 5 (50 → 45)
    end

    NV->>App: Submit số đếm SKU X = 50 lúc 9:20
    App->>DB: SELECT STOCK_TRANSACTION WHERE area=A, sku=X, created_at BETWEEN snapshot_started_at AND now()
    DB-->>App: 1 giao dịch outbound -5 xảy ra trong lúc đếm
    App->>App: Tính số kỳ vọng = counted_qty + tổng qty_delta trong lúc đếm = 50 + (-5) = 45
    App->>DB: So sánh 45 (kỳ vọng) với quantity hệ thống hiện tại 45
    Note over App: Khớp nhau, chênh lệch thực = 0, không phải do thất thoát mà do giao dịch song song đã được tính đúng

    App-->>NV: Kiểm kho hoàn tất, không có chênh lệch giả, danh sách giao dịch đã đối chiếu được đính kèm báo cáo
```
