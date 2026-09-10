# Sequence Diagram — Enhance: Reuse Diarization Result

Đây là **enhance**, flow mới xử lý yêu cầu "không xử lý lại từ đầu mỗi lần có yêu cầu tách nguồn mới" — tận dụng lại `DIARIZATION_RESULT` đã tạo từ lần xử lý ban đầu ở flow [5-enhance-separate-speaker-tracks-sqd.md](5-enhance-separate-speaker-tracks-sqd.md).

```mermaid
sequenceDiagram
    actor CustomerOrg as Khách hàng (Organization)
    participant MeetingSvc as Meeting Service
    participant DiarizationSvc as Diarization Service
    participant Storage as Recording Storage

    CustomerOrg->>MeetingSvc: Yêu cầu tách nguồn lần thứ hai (mục đích khác, ví dụ chỉ lấy 1 người nói)
    MeetingSvc->>DiarizationSvc: Check đã có DIARIZATION_RESULT cho recording này chưa

    alt đã có kết quả trung gian
        DiarizationSvc->>Storage: Fetch SPEAKER_TRACK có sẵn
        Storage-->>DiarizationSvc: Trả về track đã tách sẵn
        DiarizationSvc-->>MeetingSvc: Trả kết quả ngay, không re-process từ đầu
    else chưa có kết quả trung gian
        DiarizationSvc->>DiarizationSvc: Chạy lại flow diarization đầy đủ (xem 5-enhance-separate-speaker-tracks-sqd.md)
    end

    MeetingSvc-->>CustomerOrg: Cung cấp track theo yêu cầu
```
