# Base sequence — Xử lý mất kết nối ingest (naive)

Đây là **base**, mô tả cách hệ thống hiện tại xử lý khi streamer mất kết nối: ingest phát hiện socket đóng và kết thúc phiên live ngay lập tức, không phân biệt mất mạng tạm thời hay streamer chủ động dừng, viewer bị ngắt xem hoàn toàn. Đây là tiền đề cho enhance yêu cầu 3 — phát hiện gián đoạn trong vài giây và tự khôi phục không tạo session mới.

```mermaid
sequenceDiagram
    actor Streamer
    participant Ingest as Ingest Server
    participant Session as Stream Session Store
    actor Viewer

    Streamer--x Ingest: Mat ket noi dot ngot (rot mang)
    Ingest->>Session: UPDATE STREAM_SESSION SET status = ended, ended_at = now
    Ingest-->>Viewer: Bao stream da ket thuc, ngat playback

    Note over Streamer,Ingest: Neu streamer ket noi lai sau vai giay
    Streamer->>Ingest: Ket noi lai voi cung stream key
    Ingest->>Session: Tao STREAM_SESSION moi (session_id khac)
    Note over Session,Viewer: Vi la session moi, lich su/tuong tac cua session cu bi tach roi,<br/>viewer phai load lai trang xem tu dau
```
