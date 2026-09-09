# Enhance sequence — Ingest sync compensation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có cơ chế bù trừ độ trễ giữa các presenter. Đáp ứng **yêu cầu 2** (mỗi luồng ingest có độ trễ mạng khác nhau, hệ thống mixing phải bù trừ lệch thời gian trước khi ghép, tránh audio một presenter không khớp hành động màn hình đang hiển thị).

```mermaid
sequenceDiagram
    participant IngestA as Ingest Presenter A (mạng tốt, latency 50ms)
    participant IngestB as Ingest Presenter B (mạng yếu, latency 300ms)
    participant BufferA as SYNC_BUFFER A
    participant BufferB as SYNC_BUFFER B
    participant Mixer

    loop Liên tục
        IngestA->>BufferA: Đẩy frame, đo network_latency_ms
        IngestB->>BufferB: Đẩy frame, đo network_latency_ms
    end

    Mixer->>Mixer: Tính target_delay_ms chung = max(latency các presenter đang active) + margin an toàn
    Mixer->>BufferA: Áp target_delay_ms (đệm thêm dù mạng vốn nhanh)
    Mixer->>BufferB: Áp target_delay_ms (gần khớp latency thực của B)
    BufferA-->>Mixer: Trả frame đã căn đúng target_delay_ms
    BufferB-->>Mixer: Trả frame đã căn đúng target_delay_ms
    Mixer->>Mixer: Ghép audio/video của A và B tại cùng một mốc thời gian đã quy đổi
    Note over Mixer: Audio và hình ảnh chia sẻ màn hình của từng presenter khớp đúng nhau, dù độ trễ mạng gốc khác nhau
```
