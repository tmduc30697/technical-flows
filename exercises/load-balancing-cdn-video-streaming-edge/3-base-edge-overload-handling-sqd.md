# Base sequence — Edge overload handling

Đây là **base**, flow "Xử lý khi edge node quá tải" ở trạng thái hiện tại: khi một edge node vượt ngưỡng tải giữa lúc đang phục vụ hàng nghìn viewer, hệ thống không có phản ứng nào — cả viewer mới lẫn viewer cũ đều tiếp tục dồn vào node đó. Flow này là tiền đề cho **yêu cầu 2** (giảm tải có kiểm soát) và **yêu cầu 3** (di chuyển viewer êm) vì đây là tình huống cả hai yêu cầu đó nhắm tới xử lý.

```mermaid
sequenceDiagram
    actor NewViewer as Viewer mới
    actor ExistingViewers as Hàng nghìn viewer đang xem dở
    participant Router as Edge Router
    participant Edge as Edge Node (đang quá tải do sự kiện đông người xem)

    Note over Edge,ExistingViewers: Edge node vượt ngưỡng băng thông do một sự kiện đông người xem cùng lúc
    NewViewer->>Router: Yêu cầu xem video mới
    Router->>Router: Vẫn chọn Edge node này vì gần nhất theo địa lý
    Router-->>NewViewer: Route tới Edge node đang quá tải
    Edge-->>NewViewer: Giật/lag ngay từ đầu
    Edge-->>ExistingViewers: Tất cả viewer đang xem dở cùng bị giật/lag đồng loạt, không có biện pháp giảm tải nào được áp dụng
    Note over Edge: Không có cơ chế chuyển hướng viewer mới sang node khác, cũng không hạ bitrate viewer cũ để giảm tải có kiểm soát
```
