# Base sequence — Audio mixing

Đây là **base**, flow "Ghép audio" ở trạng thái hiện tại: mixer đơn giản cộng dồn audio của mọi ingest đang ở trạng thái unmuted, không có khái niệm "presenter chính đang trình bày". Flow này là tiền đề cho **yêu cầu 4** (tránh chồng tiếng khi nhiều presenter cùng bật mic nhầm) vì đây là nơi phát sinh lỗi chồng audio.

```mermaid
sequenceDiagram
    actor PresenterA as Presenter A (đang trình bày chính)
    actor PresenterB as Presenter B (vừa nhường lời, quên tắt mic)
    participant Mixer
    actor Viewer

    PresenterA->>Mixer: Audio unmuted, đang nói
    PresenterB->>Mixer: Bật lại mic (thao tác nhầm)
    Mixer->>Mixer: Cộng dồn mọi input audio đang unmuted, không phân biệt ai là presenter chính
    Mixer-->>Viewer: Phát ra audio chồng cả hai giọng nói cùng lúc
    Note over Viewer: Người nghe không phân biệt được ai đang trình bày chính vì không có cơ chế ưu tiên nguồn audio
```
