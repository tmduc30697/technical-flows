# Enhance sequence — Concurrent migration race (khóa key đang di chuyển)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base không xử lý race condition lúc rebalance. Đáp ứng yêu cầu 3 của đề bài: nhiều request của cùng 1 key tới gần như đồng thời ngay lúc key đó đang được di chuyển, tránh việc cả node cũ và mới cùng đếm độc lập.

```mermaid
sequenceDiagram
    actor ClientApp as Client gọi API
    participant Router as Router (tra HASH_RING)
    participant NodeOld as NODE cũ
    participant NodeNew as NODE mới
    participant Migration as COUNTER_MIGRATION (locked=true)

    Note over Migration: API key K đang được di chuyển, COUNTER_MIGRATION đánh dấu locked=true trong lúc chuyển giá trị counter

    ClientApp->>Router: Request tới ngay trong lúc K đang locked
    Router->>Migration: Kiểm tra trạng thái migration của K
    Migration-->>Router: locked=true, tạm thời mọi request của K phải đi qua NodeOld (nguồn xác thực duy nhất trong lúc lock)
    Router->>NodeOld: Forward request, tăng counter tại NodeOld
    NodeOld-->>Router: count=81/100

    Migration->>Migration: Hoàn tất chuyển giá trị counter (bao gồm cả các lần tăng vừa xảy ra trong lúc lock) sang NodeNew
    Migration->>Migration: Đặt locked=false, cập nhật HASH_RING trỏ K sang NodeNew

    ClientApp->>Router: Request tiếp theo sau khi unlock
    Router->>Migration: Kiểm tra trạng thái, thấy locked=false
    Router->>NodeNew: Forward request tới NodeNew (đã có đúng giá trị counter kế thừa)
    Note over NodeOld,NodeNew: Trong toàn bộ thời gian lock, chỉ đúng 1 node được đếm cho K tại một thời điểm, tránh đếm rải trên cả 2 node
```
