# Enhance sequence — Bắt đầu live với xác thực stream key và ABR 3 bitrate

Đây là **enhance** của flow `start-stream` đã có ở base. So với base (nhận luồng không kiểm tra key, chỉ transcode 1 chất lượng), flow này thay đổi ở 2 điểm: ingest xác thực stream key và từ chối ngay nếu sai/đã bị revoke, không tạo bất kỳ session nào cho luồng "ma" (yêu cầu 1); và transcode song song ra tối thiểu 3 mức bitrate (nguồn, trung, thấp) để phục vụ adaptive bitrate streaming, với độ trễ thêm chỉ vài giây (yêu cầu 2).

```mermaid
sequenceDiagram
    actor Streamer
    participant Ingest as Ingest Server
    participant Auth as Stream Key Validator
    participant Transcode as Transcode Workers
    participant CDN
    actor Viewer

    Streamer->>Ingest: Ket noi RTMP/SRT kem stream key
    Ingest->>Auth: Kiem tra key hop le va chua bi revoke
    alt Key sai hoac da bi revoke
        Auth-->>Ingest: Tu choi
        Ingest-->>Streamer: Dong ket noi ngay, khong tao STREAM_SESSION
    else Key hop le
        Auth-->>Ingest: OK
        Ingest->>Ingest: Tao STREAM_SESSION moi, status = live

        par Transcode dong thoi 3 muc bitrate
            Ingest->>Transcode: Tao RENDITION "source"
        and
            Ingest->>Transcode: Tao RENDITION "mid"
        and
            Ingest->>Transcode: Tao RENDITION "low"
        end
        Transcode->>CDN: Day ca 3 rendition len CDN (do tre them toi da vai giay)

        Viewer->>CDN: Yeu cau xem, client tu chon rendition theo bang thong
        CDN-->>Viewer: Tra ve rendition phu hop (ABR)
    end
```
