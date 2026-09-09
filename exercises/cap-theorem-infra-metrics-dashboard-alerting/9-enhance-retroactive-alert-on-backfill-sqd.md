# Enhance sequence — Retroactive alert on backfill

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không đánh giá lại dữ liệu gửi trễ). Đáp ứng yêu cầu thứ 4 của đề bài: dữ liệu backfill trễ vẫn phải được đánh giá lại theo ngưỡng tại đúng thời điểm xảy ra, kích hoạt cảnh báo hồi tố nếu cần thay vì bỏ qua vì "đã quá muộn".

```mermaid
sequenceDiagram
    actor Agent as Agent (vừa mất kết nối tạm thời, giờ gửi bù)
    participant Nodes as TIME_SERIES_NODE
    participant Eval as Backfill Evaluator
    participant Dashboard as Dashboard App
    actor OnCall as On-call Engineer

    Agent->>Nodes: Gửi METRIC_SAMPLE(sample_timestamp=t_cũ, is_backfill=true)
    Nodes->>Dashboard: Dữ liệu được chèn đúng vị trí theo sample_timestamp trong chuỗi thời gian
    Dashboard->>Dashboard: Biểu đồ xu hướng tự động phản ánh đúng, không tính sai lệch vì tới muộn

    Nodes->>Eval: Kích hoạt đánh giá cho mẫu backfill này
    Eval->>Eval: So sánh value với ngưỡng nguy hiểm tại đúng sample_timestamp (không phải received_at)
    alt Giá trị tại t_cũ đã vượt ngưỡng nguy hiểm
        Eval->>Eval: retroactive_alert_triggered=true
        Eval->>OnCall: Kích hoạt cảnh báo hồi tố — "Server X đã vượt ngưỡng tại t_cũ, dữ liệu vừa được xác nhận qua backfill"
        Note over OnCall: Dù đã "quá muộn" để phản ứng tức thời, on-call vẫn cần biết để kiểm tra hậu quả và tránh lặp lại
    else Giá trị tại t_cũ chưa vượt ngưỡng
        Eval->>Eval: retroactive_alert_triggered=false, không làm gì thêm
    end
```
