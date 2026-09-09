# Enhance sequence — DVR time-shift replay

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có cơ chế xem lại. Đáp ứng **yêu cầu 4** (hỗ trợ time-shift/DVR ngắn vài chục giây gần nhất, không ảnh hưởng luồng trực tiếp đang phân phối cho người khác).

```mermaid
sequenceDiagram
    actor ViewerA as Viewer đang xem live
    actor ViewerB as Viewer muốn xem lại pha bóng vừa rồi
    participant Edge as Edge Node
    participant DVR as DVR_BUFFER (edge node)

    loop Liên tục khi đang live
        Edge->>DVR: Lưu segment mới nhất, loại bỏ segment cũ ngoài window_seconds
    end

    Edge-->>ViewerA: Tiếp tục phát luồng live bình thường, không bị ảnh hưởng

    ViewerB->>Edge: Yêu cầu tua lại 20 giây gần nhất
    Edge->>DVR: Truy vấn segment trong khoảng [latest_pts - 20s, latest_pts]
    DVR-->>Edge: Trả về danh sách segment đã buffer
    Edge-->>ViewerB: Phát lại các segment đó trên một manifest/session riêng cho viewer B
    Note over ViewerA,ViewerB: Viewer A vẫn xem live liên tục, việc phục vụ time-shift cho viewer B không chiếm lại tài nguyên của luồng live chính
    ViewerB->>Edge: Quay lại xem live sau khi xem xong đoạn tua lại
    Edge-->>ViewerB: Đồng bộ viewer B về đúng video_pts hiện tại của live
```
