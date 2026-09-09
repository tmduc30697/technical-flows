# Enhance sequence — Failover điểm ingest khi sự cố hoặc khi streamer di chuyển

Đây là **enhance**, flow hoàn toàn mới so với base (base không có khái niệm nhiều điểm ingest nên không cần failover). Đáp ứng đồng thời yêu cầu 2 (điểm ingest gần nhất gặp sự cố phải tự động failover, gián đoạn không quá vài giây, không cần streamer cấu hình lại thủ công) và yêu cầu 4 (streamer di chuyển giữa buổi live khiến điểm ingest tối ưu thay đổi, không được làm rớt stream) — cả 2 đều dùng chung cơ chế reassign `INGEST_ASSIGNMENT` mà giữ nguyên `STREAM_SESSION`.

```mermaid
sequenceDiagram
    actor Streamer
    participant EndpointA as Ingest Endpoint A (dang dung)
    participant Health as Health Monitor
    participant Router as Geo Routing Service
    participant EndpointC as Ingest Endpoint C (moi)

    alt Kich ban 1: EndpointA qua tai/downtime
        Health->>Health: Phat hien EndpointA.status chuyen thanh overloaded/down
        Health->>Router: Bao su co EndpointA, yeu cau failover cho cac session dang gan EndpointA
    else Kich ban 2: Streamer di chuyen vi tri/doi mang giua buoi
        Streamer->>Router: Gui lai vi tri/do tre moi do duoc (dinh ky)
        Router->>Router: Phat hien EndpointC hien toi uu hon EndpointA cho streamer nay
    end

    Router->>EndpointC: Chuan bi nhan luong cho STREAM_SESSION hien tai (khong tao session moi)
    Router-->>Streamer: Bao dia chi ingest moi (EndpointC), giu nguyen session_id

    Streamer->>EndpointC: Ket noi lai (chuyen huong ngam, trong vong vai giay)
    EndpointC->>Router: Dong INGEST_ASSIGNMENT cu (EndpointA), tao INGEST_ASSIGNMENT moi (EndpointC, reason tuong ung)

    Note over Streamer,EndpointC: STREAM_SESSION.id khong doi trong suot qua trinh chuyen doi,<br/>viewer khong bi rot stream, chi co gian doan vai giay o buoc chuyen huong
```
