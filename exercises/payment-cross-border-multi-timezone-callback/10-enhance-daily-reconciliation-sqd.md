# Enhance sequence — Daily reconciliation

Đây là **enhance**, flow "Đối soát cuối ngày với đối tác thanh toán". So với base, mốc "cuối ngày" nay được chuẩn hóa về UTC ở cả 2 phía trước khi so khớp, và đối soát chạy riêng theo từng `currency_pair` thay vì gộp chung — so khớp cả số tiền gốc (buyer trả) lẫn số tiền quy đổi (seller nhận), phát hiện và gắn cờ các order có sai lệch tỷ giá bất thường để điều tra riêng.

```mermaid
sequenceDiagram
    participant Scheduler
    participant ReconJob as Reconciliation Job
    participant DB as Database
    participant PaymentPartner

    Scheduler->>ReconJob: Kích hoạt đối soát theo mốc 00:00 UTC
    loop với mỗi currency_pair
        ReconJob->>DB: Lấy các ORDER đã paid trong ngày UTC của đúng currency_pair này
        ReconJob->>PaymentPartner: Lấy báo cáo cuối ngày của partner, quy đổi mốc ngày của partner về UTC
        PaymentPartner-->>ReconJob: dữ liệu báo cáo theo UTC
        ReconJob->>ReconJob: So khớp buyer_amount và seller_amount cho từng order
        alt phát hiện sai lệch tỷ giá bất thường
            ReconJob->>DB: Tạo RECONCILIATION_DISCREPANCY (order_id, expected_rate, actual_rate, deviation_pct)
        end
        ReconJob->>DB: Lưu RECONCILIATION_REPORT cho currency_pair này với report_date_utc
    end
    ReconJob-->>Scheduler: Đối soát hoàn tất
    Note over ReconJob,PaymentPartner: Mỗi loại tiền tệ đối soát riêng, mốc ngày đã chuẩn hóa UTC ở cả 2 hệ thống, không còn lệch báo cáo do múi giờ
```
