# Base sequence — Presenter switch

Đây là **base**, flow "Chuyển quyền trình bày" ở trạng thái hiện tại: host bấm switch, mixer cắt ngay sang ingest của presenter mới bất kể frame mới đã kịp tới hay chưa. Flow này là tiền đề cho **yêu cầu 1** (crossfade mượt, không đen hình/đứng hình) vì đây chính là chỗ phát sinh lỗi khi lệnh chuyển đến sớm hơn dữ liệu ingest thực tế.

```mermaid
sequenceDiagram
    actor Host
    participant Mixer
    participant IngestOld as Ingest Presenter A (đang phát)
    participant IngestNew as Ingest Presenter B (được chuyển tới)
    actor Viewer

    Mixer->>Viewer: Đang phát output từ Presenter A
    Host->>Mixer: Bấm nút chuyển sang Presenter B
    Mixer->>Mixer: Ghi SWITCH_COMMAND(target=Presenter B), cập nhật current_active_presenter_id ngay lập tức
    Mixer->>IngestNew: Lấy frame hiện có của Presenter B
    Note over IngestNew: Frame mới nhất của Presenter B chưa kịp tới do độ trễ mạng riêng của presenter này
    Mixer-->>Viewer: Phát khung hình đen hoặc frame cũ/đứng hình của Presenter B trong vài trăm ms
    IngestNew-->>Mixer: Frame mới thực sự tới (trễ so với lệnh switch)
    Mixer-->>Viewer: Từ frame này trở đi mới thực sự mượt
```
