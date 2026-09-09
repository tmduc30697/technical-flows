# Enhance sequence — Range split tạo Raft group mới không mất write inflight

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base chỉ có 1 range cố định, không bao giờ split. Khi 1 range quá lớn, hệ thống tạo Raft group mới cho phần tách ra, và phải đảm bảo write đang inflight tại thời điểm split (đã gửi tới leader cũ, chưa kịp ack) không bị mất — đáp ứng đúng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor Client
    participant OldLeader as Leader Range B (trước split, key 0-1000)
    participant SplitCoord as Split Coordinator
    participant NewGroup as RAFT_GROUP mới (Range B2, key 500-1000)

    Client->>OldLeader: Write row(key=600) (inflight ngay trước lúc split)
    OldLeader->>OldLeader: Append log entry key=600 vào log Range B (chưa commit xong)

    par Song song, split được kích hoạt do Range B quá lớn
        SplitCoord->>OldLeader: Bắt đầu split tại key=500
        OldLeader->>OldLeader: Chờ toàn bộ write inflight tại thời điểm bắt đầu split commit xong (bao gồm key=600)
    end

    OldLeader-->>Client: 200 OK cho write key=600 (đã commit trước khi hoàn tất split)

    OldLeader->>NewGroup: Khởi tạo Raft group mới cho Range B2 (key 500-1000), copy state đã commit gồm key=600
    NewGroup->>NewGroup: Bầu leader riêng cho Range B2
    OldLeader->>OldLeader: Range B thu hẹp lại còn key 0-499

    Note over OldLeader,NewGroup: Write key=600 không bị mất vì được commit xong trước khi Range B2 chính thức tách ra và nhận trách nhiệm phục vụ key đó
```
