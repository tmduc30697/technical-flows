# Enhance sequence — Presenter switch (crossfade mượt)

Đây là **enhance** của flow đã có ở base "Presenter switch". So với base, mixer không cắt ngay khi nhận lệnh switch, mà tạo một `TRANSITION_EVENT`, chờ frame của presenter mới thực sự tới (dựa trên `SYNC_BUFFER`) rồi mới crossfade, đáp ứng **yêu cầu 1** (không đen hình/đứng hình/lẫn audio do lệch thời điểm giữa lệnh chuyển và ingest thực tế).

```mermaid
sequenceDiagram
    actor Host
    participant Mixer
    participant IngestOld as Ingest Presenter A (đang phát)
    participant IngestNew as Ingest Presenter B (được chuyển tới)
    participant Buffer as SYNC_BUFFER của Presenter B
    actor Viewer

    Mixer->>Viewer: Đang phát output từ Presenter A
    Host->>Mixer: Bấm nút chuyển sang Presenter B
    Mixer->>Mixer: Tạo TRANSITION_EVENT(from=A, to=B, command_issued_at=now)
    Mixer->>Buffer: Kiểm tra frame mới nhất đã buffer đủ chưa
    alt Frame của Presenter B chưa tới kịp
        Mixer-->>Viewer: Tiếp tục giữ nguyên Presenter A, không cắt sớm
        Note over Mixer,Buffer: Mixer chủ động trễ vài chục ms để chờ, tránh đen hình/đứng hình
        IngestNew-->>Buffer: Frame mới thực sự tới
    end
    Buffer-->>Mixer: Xác nhận đã có frame hợp lệ của Presenter B
    Mixer->>Mixer: Ghi applied_at=now, transition_type=crossfade
    Mixer-->>Viewer: Crossfade mượt từ Presenter A sang Presenter B, không lẫn audio hai nguồn
```
