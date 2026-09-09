# Enhance sequence — Ingest failover

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ có một nguồn ingest duy nhất nên không có khái niệm failover. Đáp ứng **yêu cầu 3** (nguồn phát dự phòng tự động chuyển sang khi nguồn chính gặp sự cố, không mất sóng giữa trận).

```mermaid
sequenceDiagram
    actor Producer as Đơn vị sản xuất
    participant Primary as Ingest Source (primary)
    participant Backup as Ingest Source (backup)
    participant Monitor as Ingest Health Monitor
    participant Transcoder
    participant Audit as AUDIT_LOG

    Producer->>Primary: Đẩy luồng chính
    Producer->>Backup: Đồng thời đẩy luồng dự phòng (song song, không dùng tới)
    Monitor->>Primary: Heartbeat check định kỳ
    Primary-->>Monitor: OK

    Note over Primary: Nguồn chính gặp sự cố kỹ thuật giữa trận
    Monitor->>Primary: Heartbeat check
    Primary--xMonitor: Không phản hồi / lỗi liên tiếp
    Monitor->>Monitor: Xác nhận primary health_status=down sau N lần check liên tiếp
    Monitor->>Transcoder: Chuyển input sang Backup
    Transcoder->>Backup: Bắt đầu nhận luồng từ backup
    Monitor->>Audit: Ghi FAILOVER_EVENT(from=primary, to=backup, reason, triggered_at)
    Note over Transcoder: Chuyển tiếp diễn ra trong vài giây, viewer chỉ thấy gián đoạn tối thiểu thay vì mất sóng hoàn toàn
```
