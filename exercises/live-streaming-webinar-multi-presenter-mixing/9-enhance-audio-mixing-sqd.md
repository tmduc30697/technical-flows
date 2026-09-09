# Enhance sequence — Audio mixing (ưu tiên đúng nguồn)

Đây là **enhance** của flow đã có ở base "Audio mixing". So với base, mixer không còn cộng dồn mọi nguồn unmuted, mà dựa vào `AUDIO_MIX_DECISION` để xác định đúng một presenter là nguồn audio chính tại mỗi thời điểm, các nguồn khác tự động bị hạ mức dù đang bật mic, đáp ứng **yêu cầu 4** (tránh chồng tiếng khi nhiều presenter cùng bật mic nhầm).

```mermaid
sequenceDiagram
    actor PresenterA as Presenter A (đang trình bày chính)
    actor PresenterB as Presenter B (vừa nhường lời, quên tắt mic)
    participant Mixer
    actor Viewer

    Mixer->>Mixer: AUDIO_MIX_DECISION hiện tại active_speaker=Presenter A
    PresenterA->>Mixer: Audio unmuted, đang nói
    PresenterB->>Mixer: Bật lại mic (thao tác nhầm)
    Mixer->>Mixer: Kiểm tra AUDIO_MIX_DECISION, phát hiện Presenter B không phải active_speaker
    Mixer->>Mixer: Tự động hạ gain audio của Presenter B về gần 0, dù đang unmuted
    Mixer-->>Viewer: Chỉ phát audio của Presenter A, không chồng tiếng
    Note over Mixer: Khi host chuyển active_speaker sang Presenter B, AUDIO_MIX_DECISION cập nhật lại và audio A tự động bị hạ tương ứng
```
