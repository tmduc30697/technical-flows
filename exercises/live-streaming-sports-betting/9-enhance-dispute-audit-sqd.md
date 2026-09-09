# Enhance sequence — Dispute audit lookup

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không ghi log đối soát tập trung. Đáp ứng **yêu cầu 5** (log đầy đủ luồng và mốc thời gian sự kiện để phục vụ đối soát khi có tranh chấp về kết quả cá cược liên quan đến độ trễ hiển thị).

```mermaid
sequenceDiagram
    actor Compliance as Nhân viên đối soát
    participant Ops as Ops Console
    participant Audit as AUDIT_LOG
    participant Timeline as EVENT_TIMELINE
    participant OddsSvc as Odds Service

    Compliance->>Ops: Mở tranh chấp cho một sự kiện cụ thể của trận đấu
    Ops->>Timeline: Tra video_pts thực tế của sự kiện (bàn thắng) tại nguồn
    Timeline-->>Ops: Trả về video_pts và occurred_at gốc
    Ops->>Audit: Truy vấn toàn bộ AUDIT_LOG liên quan match_id trong khoảng thời gian đó
    Audit-->>Ops: Trả về chuỗi log ingest, failover (nếu có), delivery từng edge, thời điểm odds_update được đẩy
    Ops->>OddsSvc: Đối chiếu target_display_pts đã dùng cho ODDS_UPDATE liên quan
    OddsSvc-->>Ops: Trả về target_display_pts đã tính tại thời điểm đó
    Ops-->>Compliance: Báo cáo đầy đủ, đủ căn cứ xác định overlay có hiển thị đúng thời điểm hay không
```
