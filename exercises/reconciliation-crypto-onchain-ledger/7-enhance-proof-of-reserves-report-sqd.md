# Enhance sequence — Xuất báo cáo proof-of-reserves từ lịch sử đối soát

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không lưu trữ lịch sử đối soát nên không có gì để báo cáo). Đáp ứng yêu cầu 5 của đề bài: toàn bộ quá trình đối soát và sai lệch phát hiện được lưu trữ minh bạch, có khả năng phục vụ báo cáo proof-of-reserves cho khách hàng/cơ quan quản lý khi được yêu cầu.

```mermaid
sequenceDiagram
    actor Requester as Khách hàng hoặc cơ quan quản lý
    participant API as Reporting API
    participant Run as RECONCILIATION_RUN
    participant Discrepancy as RECONCILIATION_DISCREPANCY
    participant Report as PROOF_OF_RESERVES_REPORT

    Requester->>API: Yêu cầu báo cáo proof-of-reserves (ví dụ tại 1 thời điểm cụ thể)
    API->>Run: Tìm RECONCILIATION_RUN gần nhất trước/tại thời điểm được yêu cầu

    alt Có RECONCILIATION_RUN phù hợp, status=matched
        API->>Discrepancy: Xác nhận không có sai lệch nào liên quan tới run này
        API->>Report: Tạo PROOF_OF_RESERVES_REPORT(reconciliation_run_id, requested_by)
        Report-->>Requester: Báo cáo xác nhận onchain_total_balance khớp ledger_total_balance tại thời điểm đó
    else RECONCILIATION_RUN có status=discrepancy_detected trong giai đoạn liên quan
        API->>Discrepancy: Lấy toàn bộ RECONCILIATION_DISCREPANCY của run đó
        API->>Report: Tạo PROOF_OF_RESERVES_REPORT kèm ghi chú minh bạch về sai lệch và tình trạng xử lý
        Report-->>Requester: Báo cáo kèm chi tiết sai lệch, không che giấu
    end

    Note over Report,Requester: Báo cáo dựa trên dữ liệu RECONCILIATION_RUN đã lưu trữ bất biến, có thể tái tạo lại bất kỳ lúc nào khi được yêu cầu kiểm toán
```
