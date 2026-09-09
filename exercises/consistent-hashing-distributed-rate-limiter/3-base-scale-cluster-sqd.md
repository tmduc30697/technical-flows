# Base sequence — Scale cluster (modulo hashing, counter bị reset)

Đây là **base**, flow khi cụm rate limiter thêm 1 node do scale tải. Vì dùng hashing kiểu modulo N (N thay đổi khi thêm node), gần như toàn bộ key bị ánh xạ lại sang node khác, và counter hiện tại không được chuyển theo — node mới coi mọi key là mới, đếm lại từ 0. Đây là vấn đề nền cho yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    participant Cluster as Rate Limiter Cluster (N node)
    participant NodeOld as NODE cũ giữ API key K (đang đếm 80/100)
    participant NodeNew as NODE mới thêm vào

    Cluster->>Cluster: Thêm 1 node mới, N tăng lên N+1
    Cluster->>Cluster: Tính lại hash mod (N+1) cho toàn bộ key
    Note over Cluster: Vì đổi từ mod N sang mod (N+1), hầu hết key đổi node phụ trách, kể cả những key không thực sự cần di chuyển
    Cluster->>NodeNew: API key K nay thuộc NODE mới theo hash mod (N+1)
    NodeNew->>NodeNew: Chưa từng thấy key K, khởi tạo RATE_LIMIT_COUNTER(K)=0/100
    Note over NodeOld,NodeNew: Counter 80/100 trên NODE cũ bị bỏ lại, không được chuyển sang, user vô tình được cấp lại 100 request mới ngay sau khi cụm scale
```
