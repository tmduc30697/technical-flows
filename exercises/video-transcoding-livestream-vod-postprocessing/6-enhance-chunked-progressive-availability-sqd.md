# Sequence Diagram — Enhance: Chunked Progressive Availability

Đây là **enhance**, flow mới phục vụ riêng yêu cầu "stream dài nhiều giờ phải xử lý theo từng đoạn, VOD sẵn sàng dần thay vì chờ toàn bộ". Đây là góc nhìn "viewer xem VOD trong lúc các chunk cuối còn đang xử lý", bổ sung cho flow [5-enhance-assemble-vod-sqd.md](5-enhance-assemble-vod-sqd.md).

```mermaid
sequenceDiagram
    actor Viewer
    participant PostProc as VOD Post-Processing Service
    participant VODStore as VOD Storage
    participant Player as VOD Player

    PostProc->>VODStore: VOD_CHUNK 1 ready (transcode_status = done)
    VODStore->>VODStore: Publish manifest với chunk 1 available

    Viewer->>Player: Open VOD
    Player->>VODStore: Request VOD manifest
    VODStore-->>Player: Manifest hiện có (chunk 1 sẵn sàng, chunk sau đang xử lý)
    Player-->>Viewer: Bắt đầu phát chunk 1

    par các chunk sau tiếp tục transcode nền
        PostProc->>VODStore: VOD_CHUNK 2 ready
        VODStore->>VODStore: Cập nhật manifest
    and viewer tiếp tục xem
        Player->>VODStore: Poll/refresh manifest khi gần hết chunk hiện tại
        VODStore-->>Player: Chunk tiếp theo đã sẵn sàng
    end

    Note over VODStore: Toàn bộ VOD_CHUNK done thì VOD.status chuyển sang fully ready
```
