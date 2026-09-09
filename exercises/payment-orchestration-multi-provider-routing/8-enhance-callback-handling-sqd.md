# Enhance sequence — Chuẩn hoá callback và map lại giao dịch qua reference_code

Đây là **enhance**, flow "Nhận callback từ nhà cung cấp" sau khi có chuẩn hoá và mã tham chiếu. So với base, flow này thay đổi ở 2 điểm: (1) mọi callback dù format gốc khác nhau đều được adapter theo từng provider chuyển về cùng 1 `normalized_status` nội bộ trước khi ghi nhận, (2) map lại `PAYMENT_REQUEST`/`TRANSACTION` gốc bằng `reference_code` đã gửi kèm lúc khởi tạo request, không còn đoán theo số tiền + thời gian.

```mermaid
sequenceDiagram
    participant ProviderA as Provider A
    participant Adapter as Provider Callback Adapter
    participant App as Orchestration Service
    participant DB as PAYMENT_REQUEST + TRANSACTION store

    ProviderA->>Adapter: Callback format riêng của A (ref=REF-789, result_code=OK)
    Adapter->>Adapter: Chuẩn hoá raw_status của A về normalized_status=success theo mapping riêng cho Provider A
    Adapter->>App: PROVIDER_CALLBACK(reference_code_received=REF-789, normalized_status=success)

    App->>DB: Tìm PAYMENT_REQUEST theo reference_code=REF-789 (khớp chính xác, không đoán theo amount+time)
    DB-->>App: Tìm thấy đúng PAYMENT_REQUEST của giao dịch #789

    App->>DB: UPDATE PAYMENT_REQUEST SET status=success, UPDATE TRANSACTION SET status=success
    App->>DB: Ghi ROUTING_LOG(action=success)

    Note over Adapter,DB: Dù 2 giao dịch khác nhau cùng gửi 500.000đ tới Provider A trong cùng khoảng thời gian, reference_code khác nhau đảm bảo callback luôn map đúng giao dịch gốc
```
