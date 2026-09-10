# Sequence Diagram — Enhance: Assemble VOD

Đây là **enhance**, flow chính hoàn toàn mới: khi stream kết thúc (kế thừa `STREAM.status = ended` từ base), hệ thống ghép các `SEGMENT` theo đúng thứ tự thời gian, xử lý segment thiếu/lỗi, rồi transcode lại ở chất lượng cao hơn bản live gốc và ánh xạ lại mốc thời gian của `LIVE_EVENT` sang VOD.

```mermaid
sequenceDiagram
    participant LiveSvc as Live Streaming Service
    participant PostProc as VOD Post-Processing Service
    participant Storage as Segment Storage
    participant Transcoder as High-Quality Transcoder
    participant VODStore as VOD Storage

    LiveSvc->>PostProc: Stream ended event (stream_id)
    PostProc->>Storage: Fetch SEGMENT list ordered by sequence_number
    Storage-->>PostProc: Segment list

    PostProc->>PostProc: Detect missing/corrupted segments
    alt segment thiếu/lỗi
        PostProc->>PostProc: Đánh dấu khoảng trống, dùng segment liền kề hợp lệ để nối liên tục
    end

    PostProc->>PostProc: Concatenate segments into continuous raw recording
    PostProc->>PostProc: Split raw recording into VOD_CHUNK theo thời lượng

    loop mỗi VOD_CHUNK (off-peak, song song)
        PostProc->>Transcoder: Transcode chunk ở chất lượng cao hơn live
        Transcoder-->>PostProc: Chunk output ready
        PostProc->>VODStore: Store transcoded chunk
    end

    PostProc->>PostProc: Remap LIVE_EVENT.occurred_at sang vod_offset_seconds theo timeline VOD mới
    PostProc->>VODStore: Mark VOD available
    VODStore-->>PostProc: Confirmed
```
