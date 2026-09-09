# Base sequence — Recording

Đây là **base**, flow "Ghi lại buổi hội thảo" ở trạng thái hiện tại: hệ thống chỉ ghi liên tục đúng luồng output cuối cùng đã mix, không lưu kèm bất kỳ mốc thời gian hay metadata nào về việc chuyển đổi giữa các presenter. Flow này là tiền đề cho **yêu cầu 5** (bản ghi phải giữ đúng trình tự chuyển đổi như đã phát trực tiếp) vì đây là chỗ thiếu thông tin để tái hiện chính xác trình tự khi phát lại.

```mermaid
sequenceDiagram
    participant Mixer
    participant Recorder
    participant Storage as Recording Storage

    loop Trong suốt buổi hội thảo
        Mixer->>Recorder: Đẩy khung hình/audio output đã mix
        Recorder->>Storage: Ghi tiếp nối vào file recording
    end
    Note over Recorder,Storage: File recording chỉ là bản sao output cuối cùng, không có mốc nào đánh dấu tại đâu đã xảy ra chuyển đổi giữa các presenter
    Note over Recorder,Storage: Khi phát lại, không thể biết chính xác trình tự và thời điểm các lần chuyển đổi đã diễn ra như lúc live
```
