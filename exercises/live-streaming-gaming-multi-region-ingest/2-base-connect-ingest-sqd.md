# Base sequence — Streamer kết nối ingest trung tâm cố định

Đây là **base**, mô tả flow streamer bắt đầu live ở trạng thái hiện tại: bất kể streamer ở khu vực địa lý nào, phần mềm phát luôn kết nối tới cùng 1 điểm ingest trung tâm được cấu hình cứng, không có bước đo độ trễ/định tuyến theo địa lý. Đây là tiền đề cho enhance yêu cầu 1 — tự động định tuyến tới ingest endpoint gần nhất.

```mermaid
sequenceDiagram
    actor StreamerAsia as Streamer o Chau A
    actor StreamerEU as Streamer o Chau Au
    participant Central as Central-PoP (dia chi co dinh)

    Note over StreamerAsia,StreamerEU: Ca 2 streamer deu duoc cau hinh cung 1 dia chi ingest trung tam

    StreamerAsia->>Central: Ket noi RTMP toi Central-PoP (do tre mang cao do khoang cach xa)
    Central-->>StreamerAsia: Chap nhan, bat dau STREAM_SESSION

    StreamerEU->>Central: Ket noi RTMP toi Central-PoP (do tre mang thap hon vi gan hon)
    Central-->>StreamerEU: Chap nhan, bat dau STREAM_SESSION

    Note over Central: Khong co co che chon diem ingest gan nhat,<br/>streamer o xa luon chiu do tre ingest cao hon streamer o gan
```
