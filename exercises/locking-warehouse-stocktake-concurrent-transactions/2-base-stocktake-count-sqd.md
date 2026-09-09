# Base sequence — Stocktake count (không khóa, không đối chiếu, gây chênh lệch giả)

Đây là **base**, flow kiểm kho hiện tại: nhân viên đếm thủ công rồi submit số đếm, hệ thống không chặn giao dịch xuất/nhập nào khác đang chạy song song ở cùng khu vực, cũng không ghi nhận mốc thời gian bắt đầu đếm để đối chiếu sau này. Đây chính là kịch bản mô tả ở yêu cầu 1 của đề bài — nhân viên đếm 50 đơn vị SKU X lúc 9:00, đúng lúc có phiếu xuất 5 đơn vị chạy song song, số đếm 50 bị ghi đè trực tiếp dù hệ thống lúc đó đã là 45, gây báo cáo chênh lệch giả.

```mermaid
sequenceDiagram
    actor NV as Nhân viên kiểm kho
    participant App as Stocktake App
    participant DB as Database
    participant OMS as Hệ thống xử lý đơn hàng

    NV->>App: Bắt đầu kiểm kho khu vực A lúc 9:00
    App->>DB: INSERT STOCKTAKE (area=A, status=in_progress)
    Note over App,DB: Không khóa area A, không ghi timestamp để đối chiếu sau

    NV->>NV: Đếm thủ công SKU X tại khu vực A, đếm được 50

    par Song song trong lúc đang đếm
        OMS->>DB: Phiếu xuất kho 5 đơn vị SKU X cho đơn giao hàng
        DB->>DB: UPDATE INVENTORY_ITEM SET quantity = quantity - 5 (50 → 45)
    end

    NV->>App: Submit số đếm SKU X = 50
    App->>DB: UPDATE INVENTORY_ITEM SET quantity = 50 WHERE area=A, sku=X
    Note over DB: Ghi đè trực tiếp, không kiểm tra system_qty tại thời điểm submit đã đổi thành 45
    DB-->>App: OK

    App-->>NV: Kiểm kho hoàn tất, chênh lệch báo cáo = 50 - 45(gốc) = +5
    Note over App,DB: Chênh lệch +5 này là giả, thực chất do phiếu xuất chạy song song, không phải thất thoát thực, nhưng hệ thống không có cách phân biệt
```
