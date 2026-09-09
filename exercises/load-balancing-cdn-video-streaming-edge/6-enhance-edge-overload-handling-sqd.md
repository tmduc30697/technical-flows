# Enhance sequence — Edge overload handling (giảm tải có kiểm soát)

Đây là **enhance** của flow đã có ở base "Edge overload handling". So với base, khi node vượt ngưỡng tải, viewer mới bị chuyển hướng ngay sang node lân cận, còn viewer đang xem dở được hạ bitrate có chọn lọc thay vì để tất cả cùng giật, đáp ứng **yêu cầu 2** (chuyển hướng viewer mới sang node khác, chiến lược giảm tải có kiểm soát cho viewer đang xem dở).

```mermaid
sequenceDiagram
    actor NewViewer as Viewer mới
    actor ExistingViewers as Viewer đang xem dở trên node quá tải
    participant Router as Edge Router
    participant Edge as Edge Node (đang quá tải)
    participant EdgeAlt as Edge Node lân cận (còn tải)
    participant Controller as Load Shedding Controller

    Note over Edge: current_viewer_count vượt ngưỡng do sự kiện đông người xem
    Controller->>Edge: Phát hiện vượt ngưỡng tải qua LOAD_METRIC_UPDATE liên tục
    Controller->>Router: Đánh dấu Edge node này tạm ngừng nhận viewer mới

    NewViewer->>Router: Yêu cầu xem video mới
    Router->>Router: Thấy Edge node đang bị đánh dấu quá tải, không route tới nữa
    Router-->>NewViewer: Route sang EdgeAlt lân cận

    Controller->>Controller: Chọn một phần ExistingViewers để giảm tải có kiểm soát (không phải toàn bộ)
    Controller->>Edge: Ghi BITRATE_DEGRADATION_EVENT, hạ bitrate của các session được chọn
    Edge-->>ExistingViewers: Một phần viewer bị hạ chất lượng bitrate có kiểm soát, phần còn lại vẫn giữ bitrate cũ
    Note over ExistingViewers: Tổng băng thông node giảm về dưới ngưỡng, tránh tình trạng tất cả cùng giật/lag đồng loạt
```
