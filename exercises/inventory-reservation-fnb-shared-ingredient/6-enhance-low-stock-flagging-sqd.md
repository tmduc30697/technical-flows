# Enhance sequence — Đánh dấu "sắp hết" theo thời gian gần thực trên mọi kênh

Đây là **enhance**, flow hoàn toàn mới so với base (base chưa có khái niệm ngưỡng cảnh báo hay đồng bộ trạng thái món qua kênh). Đáp ứng yêu cầu 3 trong đề bài: ngay sau khi 1 giao dịch trừ tồn khiến nguyên liệu xuống dưới ngưỡng cảnh báo, các món liên quan phải được đánh dấu "sắp hết"/ẩn trên mọi kênh gần như tức thời, tránh khách tiếp tục đặt món chắc chắn bị từ chối.

```mermaid
sequenceDiagram
    participant Stock as Ingredient Stock (DB)
    participant Reservation as Reservation Service
    participant Broadcaster as Stock Sync Broadcaster
    participant AppChannel as Kenh App
    participant InstoreChannel as Kenh Tai cho
    participant PartnerChannel as Kenh Doi tac giao do an

    Reservation->>Stock: Sau khi tru tru "thit bo" thanh cong, doc lai stock_quantity moi
    Stock-->>Reservation: stock_quantity = 150g (duoi low_stock_threshold cho 3 suat)
    Reservation->>Reservation: Danh dau ingredient.is_low_stock = true

    Reservation->>Broadcaster: Publish su kien low-stock cho "thit bo" va cac menu_item lien quan (Pho bo, Bun bo)
    par Phan phoi gan thoi gian thuc toi tat ca kenh
        Broadcaster->>AppChannel: Cap nhat is_available = false cho Pho bo, Bun bo
    and
        Broadcaster->>InstoreChannel: Cap nhat is_available = false cho Pho bo, Bun bo
    and
        Broadcaster->>PartnerChannel: Cap nhat is_available = false cho Pho bo, Bun bo
    end

    Note over AppChannel,PartnerChannel: Menu tren ca 3 kenh an/danh dau sap het gan nhu dong thoi,<br/>tranh khach dat mon chac chan bi tu choi vi nguyen lieu vua duoc don khac dung het vai giay truoc
```
