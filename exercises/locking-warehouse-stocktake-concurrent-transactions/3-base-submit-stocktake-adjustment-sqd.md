# Base sequence — Submit stocktake adjustment (không version check, ghi đè mù)

Đây là **base**, flow nộp kết quả điều chỉnh sau khi đếm xong: nhân viên xác nhận số liệu cuối cùng và hệ thống ghi đè trực tiếp vào `INVENTORY_ITEM.quantity`, không kiểm tra số liệu tồn kho đã thay đổi kể từ lúc bắt đầu đếm hay chưa. Đây là tiền đề cho yêu cầu 4 của đề bài — khi có giao dịch nhập kho mới hoàn tất đúng lúc nhân viên đang chờ submit, việc ghi đè mù sẽ làm mất luôn số lượng hàng mới nhập.

```mermaid
sequenceDiagram
    actor NV as Nhân viên kiểm kho
    participant App as Stocktake App
    participant DB as Database
    participant OMS as Hệ thống nhập kho

    Note over NV,App: Đang hoàn tất kiểm kho khu vực A, chuẩn bị submit điều chỉnh SKU X

    OMS->>DB: Phiếu nhập kho 20 đơn vị SKU X vào khu vực A vừa hoàn tất
    DB->>DB: UPDATE INVENTORY_ITEM SET quantity = quantity + 20 (45 → 65)

    NV->>App: Submit điều chỉnh cuối cùng, SKU X = 50 (theo số đã đếm từ đầu)
    App->>DB: UPDATE INVENTORY_ITEM SET quantity = 50 WHERE area=A, sku=X
    Note over App,DB: Không đọc lại quantity hiện tại (65), không so sánh gì cả trước khi ghi đè
    DB-->>App: OK

    App-->>NV: Điều chỉnh thành công, quantity=50
    Note over DB: 20 đơn vị vừa nhập kho bị mất khỏi hệ thống ngay lập tức, không ai được cảnh báo
```
