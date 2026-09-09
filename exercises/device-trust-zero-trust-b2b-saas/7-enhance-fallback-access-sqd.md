# Enhance sequence — Fallback VPN tạm thời có xác minh thủ công bởi IT

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp khẩn cấp: nhân viên cần truy cập gấp từ thiết bị không đạt chuẩn posture (hoặc chưa kịp đăng ký MDM). Đáp ứng yêu cầu 5: luồng dự phòng có kiểm soát, được ghi log đầy đủ và giới hạn thời gian, không phải là lối tắt bỏ qua Zero Trust vĩnh viễn.

```mermaid
sequenceDiagram
    actor User
    participant App as Internal Tool
    actor IT as IT Staff
    participant VPN as VPN Gateway
    participant DB as Database

    User->>App: Login từ thiết bị không đạt chuẩn, bị từ chối (như flow login)
    User->>App: Bấm "Yêu cầu truy cập khẩn cấp"
    App->>DB: INSERT FALLBACK_ACCESS_REQUEST (user_id, reason, status=pending, requested_at)
    App-->>User: "Yêu cầu đã gửi tới IT, vui lòng chờ xác minh"

    App->>IT: Thông báo có yêu cầu fallback access mới cần duyệt

    IT->>App: Xem yêu cầu, gọi điện/video xác minh danh tính user thủ công
    IT->>App: Duyệt yêu cầu, cấp thời hạn (vd 4 giờ)

    App->>DB: UPDATE FALLBACK_ACCESS_REQUEST SET status=approved, approved_by_it_user_id=IT, expires_at=now()+4h
    App->>VPN: Cấp VPN grant tạm thời (vpn_grant_id) giới hạn thời gian cho user
    VPN-->>App: Grant tạo thành công
    App->>DB: UPDATE FALLBACK_ACCESS_REQUEST SET vpn_grant_id=...

    App-->>User: Truy cập được qua VPN tạm thời trong thời hạn được cấp

    Note over App,DB: Toàn bộ thao tác (ai yêu cầu, ai duyệt, thời điểm, thời hạn) được log đầy đủ trong FALLBACK_ACCESS_REQUEST để audit sau này

    App->>VPN: Sau khi hết expires_at, tự động thu hồi VPN grant
    VPN-->>App: Grant đã bị revoke
    App->>DB: UPDATE FALLBACK_ACCESS_REQUEST SET status=expired
```
