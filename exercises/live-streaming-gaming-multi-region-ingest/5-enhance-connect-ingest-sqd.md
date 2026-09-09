# Enhance sequence — Định tuyến streamer tới ingest endpoint gần nhất

Đây là **enhance** của flow `connect-ingest` đã có ở base. So với base (luôn kết nối tới 1 điểm ingest trung tâm cố định), flow này thay đổi ở chỗ: hệ thống đo độ trễ mạng/vị trí địa lý của streamer, tự động chọn `INGEST_ENDPOINT` gần nhất còn khoẻ mạnh trong đúng `REGION` phù hợp, không ép cứng 1 endpoint trung tâm — đáp ứng yêu cầu 1.

```mermaid
sequenceDiagram
    actor Streamer
    participant Router as Geo Routing Service
    participant EndpointA as Ingest Endpoint (vung gan nhat)
    participant EndpointB as Ingest Endpoint (vung khac)

    Streamer->>Router: Yeu cau bat dau live, gui vi tri/do tre do duoc toi cac PoP
    Router->>Router: Doc danh sach INGEST_ENDPOINT theo REGION, loc status = healthy
    Router->>Router: Chon endpoint co do tre thap nhat cho streamer nay

    Router-->>Streamer: Tra ve dia chi EndpointA (gan nhat)
    Streamer->>EndpointA: Ket noi RTMP/SRT
    EndpointA->>Router: Tao STREAM_SESSION + INGEST_ASSIGNMENT (reason = nearest)
    EndpointA-->>Streamer: Xac nhan bat dau live

    Note over EndpointB: Khong tham gia, chi la du phong cho cac streamer o vung khac hoac khi EndpointA gap su co
```
