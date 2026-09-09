# Enhance sequence — Seamless viewer migration

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có cơ chế di chuyển viewer giữa các edge node. Đáp ứng **yêu cầu 3** (chuyển viewer từ edge quá tải sang node khác giữa lúc đang phát mà không làm gián đoạn luồng người dùng nhận biết được, tương tự đổi kênh CDN chứ không phải ngắt và load lại từ đầu).

```mermaid
sequenceDiagram
    actor Viewer
    participant Player as Video Player (client)
    participant Edge as Edge Node quá tải (nguồn cũ)
    participant EdgeAlt as Edge Node lân cận (đích mới)
    participant Controller as Load Shedding Controller

    Controller->>Controller: Chọn Viewer này để migrate sang EdgeAlt (một trong các viewer đang xem dở trên node quá tải)
    Controller->>Player: Gửi tín hiệu "chuẩn bị chuyển node", kèm địa chỉ EdgeAlt
    Player->>Player: Ghi nhận playback_position_seconds hiện tại
    Player->>EdgeAlt: Mở kết nối mới, yêu cầu tiếp tục phát từ đúng playback_position_seconds
    EdgeAlt-->>Player: Trả về segment bắt đầu đúng vị trí yêu cầu
    Player->>Player: Buffer segment mới song song trong khi vẫn phát nốt buffer cũ từ Edge
    Player->>Player: Chuyển nguồn phát sang EdgeAlt tại đúng thời điểm buffer mới sẵn sàng
    Player-->>Viewer: Video tiếp tục phát liền mạch, không giật/không load lại từ đầu
    Controller->>Controller: Ghi MIGRATION_EVENT(from=Edge, to=EdgeAlt, playback_position_seconds, reason=overload)
    Player->>Edge: Đóng kết nối cũ sau khi đã chuyển hẳn sang EdgeAlt
```
