# Sequence - Base: Xử lý pipeline video (mỗi job 1 trace riêng)

Đây là flow **base**: sau khi upload, video đi qua kiểm duyệt, rồi transcode song song 3 độ phân giải, rồi tạo thumbnail và publish. Mỗi job tự sinh trace riêng của nó trên worker mà nó chạy, không có trace gốc chung, nên không thể nhìn thấy quan hệ song song thật giữa 3 job transcode — trên dashboard chúng hiện ra như 3 trace độc lập, không rõ có chạy cùng lúc hay không. Đây là tiền đề cho yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    participant User as Người dùng
    participant API as Upload API
    participant Queue as Job Queue
    participant Mod as Worker Kiểm duyệt
    participant T1 as Worker Transcode 480p
    participant T2 as Worker Transcode 720p
    participant T3 as Worker Transcode 1080p
    participant Thumb as Worker Thumbnail
    participant Pub as Worker Publish

    User->>API: Upload video
    API->>Queue: Enqueue job kiểm duyệt
    Queue->>Mod: Chạy job kiểm duyệt
    Note over Mod: Trace #1 (chỉ của job này)
    Mod-->>Queue: Kiểm duyệt đạt

    par Transcode song song 3 độ phân giải
        Queue->>T1: Enqueue job transcode 480p
        Note over T1: Trace #2 (độc lập, không liên kết trace #1)
        T1-->>Queue: Xong 480p
    and
        Queue->>T2: Enqueue job transcode 720p
        Note over T2: Trace #3 (độc lập)
        T2-->>Queue: Xong 720p
    and
        Queue->>T3: Enqueue job transcode 1080p
        Note over T3: Trace #4 (độc lập)
        T3-->>Queue: Xong 1080p
    end
    Note over Queue: Cả 3 trace #2, #3, #4 không có liên kết nào với nhau hay với trace #1, dashboard không biết chúng thuộc cùng 1 video

    Queue->>Thumb: Enqueue job tạo thumbnail
    Note over Thumb: Trace #5 (độc lập)
    Thumb-->>Queue: Xong thumbnail

    Queue->>Pub: Enqueue job publish
    Note over Pub: Trace #6 (độc lập)
    Pub-->>Queue: Video đã publish
    Queue-->>API: Pipeline hoàn tất
```
