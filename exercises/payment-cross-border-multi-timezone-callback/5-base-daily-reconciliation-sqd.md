# Base sequence — Daily reconciliation

Đây là **base**, flow "Đối soát cuối ngày với đối tác thanh toán" — chọn flow này vì đây chính là flow bị ảnh hưởng trực tiếp bởi 2 vấn đề enhance nêu: mốc "cuối ngày" ở base tính theo giờ local của hệ thống (không chuẩn hóa múi giờ với đối tác), và dữ liệu bị gộp chung mọi loại tiền tệ thay vì tách riêng.

```mermaid
sequenceDiagram
    participant Scheduler
    participant ReconJob as Reconciliation Job
    participant DB as Database
    participant PaymentPartner

    Scheduler->>ReconJob: Kích hoạt đối soát lúc 00:00 giờ local của hệ thống
    ReconJob->>DB: Lấy toàn bộ ORDER đã paid trong ngày (gộp mọi currency_pair)
    ReconJob->>PaymentPartner: Lấy báo cáo cuối ngày của partner (theo múi giờ của partner)
    PaymentPartner-->>ReconJob: dữ liệu báo cáo cuối ngày
    ReconJob->>ReconJob: So khớp tổng số tiền giữa 2 bên
    Note over ReconJob,PaymentPartner: Mốc "cuối ngày" của 2 hệ thống lệch nhau do khác múi giờ, dễ làm sai lệch báo cáo
    Note over ReconJob,DB: Gộp chung mọi loại tiền tệ, khó phát hiện đơn hàng nào bị lệch tỷ giá bất thường
    ReconJob-->>Scheduler: Đối soát hoàn tất
```
