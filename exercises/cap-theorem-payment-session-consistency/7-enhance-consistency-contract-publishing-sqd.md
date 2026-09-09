# Enhance sequence — Consistency contract publishing

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không tài liệu hoá gì về consistency). Đáp ứng yêu cầu thứ 5 của đề bài: viết rõ và công bố mức consistency guarantee của từng endpoint để team khác (API consumer) dùng đúng cách, tránh giả định sai.

```mermaid
sequenceDiagram
    actor Platform as Platform Team
    participant Policy as CONSISTENCY_POLICY store
    participant Contract as CONSISTENCY_CONTRACT store
    actor Consumer as Team tiêu thụ API khác

    Platform->>Policy: Lấy toàn bộ endpoint và mode đã phân loại (AP/CP)
    loop Với mỗi endpoint
        Platform->>Contract: Viết CONSISTENCY_CONTRACT (guarantee_description rõ ràng, vd "read-session: AP, có thể trễ tới vài trăm ms" hoặc "confirm-payment: CP, có thể bị từ chối khi thiếu quorum, không bao giờ trả kết quả sai")
        Platform->>Contract: published_to_consumers=true
    end

    Consumer->>Contract: Tra cứu guarantee trước khi tích hợp endpoint
    Contract-->>Consumer: Trả đúng mức consistency guarantee của endpoint đó
    Consumer->>Consumer: Thiết kế retry/xử lý lỗi phía mình đúng theo guarantee (vd biết trước confirm-payment có thể trả lỗi tạm thời và cần retry có kiểm soát, không giả định luôn thành công)
```
