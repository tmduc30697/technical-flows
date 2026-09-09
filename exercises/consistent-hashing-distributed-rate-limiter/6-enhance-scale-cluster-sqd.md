# Enhance sequence — Scale cluster (chuyển counter đúng giá trị, không reset)

Đây là **enhance**, cùng flow "scale-cluster" đã có ở base nhưng nay thay đổi: nhờ consistent hashing với virtual node, chỉ 1 phần nhỏ key bị dịch chuyển khi thêm node, và giá trị counter hiện tại của các key đó được chuyển nguyên vẹn sang node mới thay vì reset về 0. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    participant Ring as HASH_RING
    participant NodeOld as NODE cũ giữ API key K (đang đếm 80/100)
    participant NodeNew as NODE mới thêm vào
    participant Migration as COUNTER_MIGRATION

    Ring->>Ring: Thêm NODE mới, chỉ các virtual node trong 1 dải hash nhỏ được gán lại cho node mới
    Ring->>Migration: API key K rơi vào dải bị dịch chuyển, tạo COUNTER_MIGRATION(from=NodeOld, to=NodeNew, status=in_progress)

    Migration->>NodeOld: Đọc RATE_LIMIT_COUNTER(K) hiện tại (80/100, còn hiệu lực trong window hiện tại)
    NodeOld-->>Migration: count=80, window_start=hiện tại
    Migration->>NodeNew: Ghi RATE_LIMIT_COUNTER(K) với đúng count=80 và window_start giữ nguyên
    Migration->>Ring: Cập nhật ring_version, xác nhận K chính thức thuộc NodeNew
    Migration->>NodeOld: Xóa counter cục bộ của K sau khi xác nhận đã chuyển thành công

    Note over NodeOld,NodeNew: User vẫn chỉ còn 20 request khả dụng trong window hiện tại, không được cấp lại 100 request mới do rebalance
```
