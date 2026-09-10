# Sequence Diagram — Enhance: Separate Speaker Tracks

Đây là **enhance**, flow mới hoàn toàn: tách audio/video theo từng người nói khi có yêu cầu, đồng bộ chính xác đến từng khung hình với video chung để tránh audio người này chồng lấn/lệch với hình ảnh người khác. Kết quả `DIARIZATION_RESULT` được lưu lại để flow [6-enhance-reuse-diarization-sqd.md](6-enhance-reuse-diarization-sqd.md) tái sử dụng.

```mermaid
sequenceDiagram
    actor CustomerOrg as Khách hàng (Organization)
    participant MeetingSvc as Meeting Service
    participant DiarizationSvc as Diarization Service
    participant Storage as Recording Storage

    CustomerOrg->>MeetingSvc: Yêu cầu tách audio/video theo từng người nói
    MeetingSvc->>DiarizationSvc: Request diarization cho RECORDING (multi-track audio)

    DiarizationSvc->>Storage: Fetch recording + raw audio track theo từng PARTICIPANT
    DiarizationSvc->>DiarizationSvc: Đồng bộ từng track với video chung, tính sync_offset_ms chính xác đến khung hình
    DiarizationSvc->>DiarizationSvc: Tách SPEAKER_TRACK riêng cho từng participant

    DiarizationSvc->>Storage: Store DIARIZATION_RESULT + SPEAKER_TRACK(s)
    DiarizationSvc-->>MeetingSvc: Diarization completed
    MeetingSvc-->>CustomerOrg: Cung cấp bản ghi tách riêng theo từng người nói
```
