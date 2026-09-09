# Enhance sequence — Cảnh báo khẩn khi ledger vượt tài sản thực on-chain

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có cơ chế phát hiện hay cảnh báo sai lệch). Đáp ứng yêu cầu 3 của đề bài: phát hiện ledger nội bộ vượt quá tài sản thực trên ví on-chain (dấu hiệu nghiêm trọng) và cảnh báo khẩn ngay lập tức, không chờ báo cáo đối soát định kỳ theo lịch thông thường.

```mermaid
sequenceDiagram
    participant Run as RECONCILIATION_RUN
    participant Discrepancy as RECONCILIATION_DISCREPANCY
    participant Detector as Discrepancy Severity Classifier
    participant Alert as URGENT_ALERT
    actor OnCall as Đội trực (security/finance)

    Run->>Discrepancy: Ghi RECONCILIATION_DISCREPANCY(discrepancy_type, amount)
    Discrepancy->>Detector: Phân loại mức độ nghiêm trọng

    alt discrepancy_type=ledger_exceeds_onchain, tức tổng ledger lớn hơn tổng tài sản thực có
        Detector->>Discrepancy: Đánh dấu severity=critical
        Detector->>Alert: Tạo URGENT_ALERT ngay lập tức, không chờ chu kỳ báo cáo định kỳ tiếp theo
        Alert-->>OnCall: Cảnh báo khẩn, kèm asset_type và amount chênh lệch
        Note over OnCall,Alert: Dấu hiệu này có thể là lỗi hệ thống hoặc gian lận nội bộ, cần đóng băng rút tiền loại tài sản liên quan để điều tra trước khi có thêm thiệt hại
    else discrepancy_type=onchain_exceeds_ledger hoặc minor_variance
        Detector->>Discrepancy: Đánh dấu severity=warning
        Note over Detector: Tài sản thừa ra hoặc sai lệch nhỏ không đe dọa khả năng chi trả ngay lập tức, xử lý theo quy trình đối soát định kỳ bình thường
    end

    OnCall->>Alert: Xác nhận đã tiếp nhận, cập nhật status=acknowledged
    OnCall->>Alert: Sau khi điều tra xong, cập nhật status=resolved
```
