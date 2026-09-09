# Enhance sequence — Stocktake count (khóa cứng khu vực, giới hạn phạm vi + thời gian)

Đây là **enhance** của flow `stocktake-count` đã có ở base. So với base (không khóa, ghi đè trực tiếp gây chênh lệch giả), nay hệ thống khóa cứng đúng khu vực A đang kiểm (không khóa toàn kho, không khóa theo SKU), chặn mọi giao dịch xuất/nhập vào khu vực đó trong lúc đếm, và tự động cảnh báo/giải phóng khóa nếu vượt thời gian tối đa cho phép. Đáp ứng yêu cầu 1 (chọn phương án khóa cứng) và yêu cầu 2 (giới hạn phạm vi + thời gian + cảnh báo) của đề bài.

```mermaid
sequenceDiagram
    actor NV as Nhân viên kiểm kho
    participant App as Stocktake App
    participant DB as Database
    participant OMS as Hệ thống xử lý đơn hàng
    participant Scheduler as Job cảnh báo hết hạn khóa

    NV->>App: Bắt đầu kiểm kho khu vực A lúc 9:00
    App->>DB: UPDATE WAREHOUSE_AREA SET lock_status=locked, locked_by=NV, lock_started_at=9:00, lock_expires_at=9:30 WHERE id=A
    DB-->>App: OK, khóa chỉ áp dụng cho area A

    OMS->>DB: Phiếu xuất kho 5 đơn vị SKU X cho khu vực A
    DB-->>OMS: Từ chối, "khu vực A đang được kiểm kho, thử lại sau"
    Note over OMS: Giao dịch vào khu vực khác (vd khu vực B) vẫn chạy bình thường, không bị ảnh hưởng

    NV->>NV: Đếm thủ công SKU X tại khu vực A, đếm được 50

    Scheduler->>DB: Kiểm tra định kỳ các area đang locked quá lock_expires_at
    Note over Scheduler: 9:25, còn 5 phút tới hạn, gửi cảnh báo cho NV và quản lý kho

    NV->>App: Submit số đếm SKU X = 50 lúc 9:20
    App->>DB: UPDATE INVENTORY_ITEM SET quantity=50, version=version+1 WHERE area=A, sku=X
    App->>DB: UPDATE WAREHOUSE_AREA SET lock_status=free, locked_by=null WHERE id=A
    DB-->>App: OK, giải phóng khóa khu vực A

    App-->>NV: Kiểm kho hoàn tất, không có giao dịch nào bị bỏ sót vì đã bị chặn suốt thời gian khóa
```
