# Enhance sequence — Submit stocktake adjustment (version check + audit trail đầy đủ)

Đây là **enhance** của flow `submit-stocktake-adjustment` đã có ở base. So với base (ghi đè mù, mất luôn hàng vừa nhập), nay hệ thống dùng `INVENTORY_ITEM.version` để phát hiện số liệu tồn kho đã thay đổi kể từ lúc bắt đầu đếm, từ chối áp trực tiếp mà yêu cầu nhân viên xác nhận lại phần chênh lệch, đồng thời ghi đầy đủ audit trail. Đáp ứng yêu cầu 4 và yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor NV as Nhân viên kiểm kho
    participant App as Stocktake App
    participant DB as Database
    participant OMS as Hệ thống nhập kho

    Note over NV,App: Bắt đầu đếm SKU X lúc 9:00, quantity=45, version=12, đếm được 50

    OMS->>DB: Phiếu nhập kho 20 đơn vị SKU X vào khu vực A vừa hoàn tất
    DB->>DB: UPDATE INVENTORY_ITEM SET quantity=65, version=13 WHERE area=A, sku=X
    DB->>DB: INSERT STOCK_TRANSACTION (type=inbound, qty_delta=+20)

    NV->>App: Submit điều chỉnh cuối cùng, SKU X = 50, version đã đọc lúc bắt đầu = 12
    App->>DB: UPDATE INVENTORY_ITEM SET quantity=50, version=13 WHERE area=A, sku=X AND version=12
    DB-->>App: 0 row affected, version hiện tại đã là 13
    Note over App: Phát hiện số liệu đã đổi, từ chối áp trực tiếp

    App->>DB: SELECT STOCK_TRANSACTION WHERE area=A, sku=X, created_at trong lúc đếm
    DB-->>App: 1 giao dịch inbound +20 lúc 9:15
    App-->>NV: Yêu cầu xác nhận lại, "hệ thống ghi nhận thêm 20 đơn vị nhập kho trong lúc đếm, số kỳ vọng mới = 50 + 20 = 70, xác nhận điều chỉnh về 70?"

    NV->>App: Xác nhận điều chỉnh = 70 (confirmed_by_staff=true)
    App->>DB: UPDATE INVENTORY_ITEM SET quantity=70, version=14 WHERE area=A, sku=X AND version=13
    DB-->>App: OK
    App->>DB: INSERT STOCKTAKE_ITEM audit (system_qty_before=45, counted_qty=50, system_qty_at_submit=65, transactions_during_count=[phiếu nhập +20], adjusted_qty=70, confirmed_by_staff=true)
    DB-->>App: OK

    App-->>NV: Điều chỉnh hoàn tất, toàn bộ số liệu trước/đếm/sau và giao dịch liên quan đã được ghi lại để truy vết sau này
```
