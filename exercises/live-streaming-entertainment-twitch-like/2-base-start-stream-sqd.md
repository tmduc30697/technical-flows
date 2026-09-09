# Base sequence — Bắt đầu live: ingest, transcode 1 bitrate, phân phối CDN

Đây là **base**, mô tả flow bắt đầu live ở trạng thái hiện tại: ingest server nhận luồng RTMP/SRT mà không kiểm tra tính hợp lệ của stream key, transcode ra đúng 1 mức chất lượng (không có ABR), rồi đẩy thẳng qua CDN cho viewer. Đây là tiền đề để enhance thêm xác thực key (yêu cầu 1) và ABR nhiều bitrate (yêu cầu 2).

```mermaid
sequenceDiagram
    actor Streamer
    participant Ingest as Ingest Server
    participant Transcode as Transcode Worker
    participant CDN
    actor Viewer

    Streamer->>Ingest: Ket noi RTMP/SRT kem stream key
    Note over Ingest: Khong kiem tra key hop le hay khong, cu nhan luong
    Ingest->>Transcode: Forward luong goc
    Transcode->>Transcode: Transcode ra 1 muc chat luong duy nhat
    Transcode->>CDN: Day luong da transcode len CDN
    Viewer->>CDN: Yeu cau xem stream
    CDN-->>Viewer: Tra ve luong (1 chat luong co dinh, khong thich ung bang thong)
```
