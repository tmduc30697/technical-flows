# Enhance sequence — Recording (giữ đúng trình tự chuyển đổi)

Đây là **enhance** của flow đã có ở base "Recording". So với base, recorder không chỉ ghi output thô mà còn gắn kèm toàn bộ `TRANSITION_EVENT` (bao gồm cả các khoảng crossfade ngắn) vào bản ghi, đáp ứng **yêu cầu 5** (bản ghi phải giữ đúng trình tự chuyển đổi giữa các nguồn như đã phát trực tiếp, kể cả các khoảng chuyển tiếp ngắn, để người xem lại không nhầm lẫn thứ tự trình bày).

```mermaid
sequenceDiagram
    participant Mixer
    participant Recorder
    participant Storage as Recording Storage
    actor Compliance as Người xem lại sau này

    loop Trong suốt buổi hội thảo
        Mixer->>Recorder: Đẩy khung hình/audio output đã mix
        Mixer->>Recorder: Đẩy kèm TRANSITION_EVENT khi có chuyển đổi (from, to, applied_at, transition_type)
        Recorder->>Storage: Ghi nối tiếp media, đồng thời ghi timeline TRANSITION_EVENT tương ứng vào RECORDING
    end
    Note over Recorder,Storage: Kể cả các đoạn crossfade ngắn giữa hai presenter cũng được đánh dấu đúng thời điểm trong timeline, không bị gộp mất
    Compliance->>Storage: Phát lại RECORDING sau này
    Storage-->>Compliance: Phát đúng thứ tự và thời điểm các lần chuyển đổi như đã diễn ra lúc live, không gây nhầm lẫn thứ tự trình bày
```
