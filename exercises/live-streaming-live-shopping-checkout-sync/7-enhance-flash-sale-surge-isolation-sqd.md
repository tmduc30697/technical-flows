# Enhance sequence — Cách ly tải đột biến checkout khỏi pipeline video

Đây là **enhance**, flow hoàn toàn mới so với base (base không có cơ chế cách ly, mọi thứ xử lý đồng bộ chung 1 luồng). Đáp ứng yêu cầu 4 trong đề bài: thời điểm mở bán flash sale khiến viewer tăng đột biến trong vài giây, hệ thống phải cách ly để tải đột biến phía checkout không ảnh hưởng ngược lại tới độ ổn định luồng video đang phát cho các viewer khác.

```mermaid
sequenceDiagram
    actor SellerLive as Nguoi ban
    participant Source as Broadcast Event Recorder
    participant Pipeline as Ingest/Transcode/CDN Pipeline
    participant Queue as Checkout Request Queue (buffer rieng)
    participant Checkout as Checkout Service (auto-scale rieng)
    participant Stock as Product Stock (DB)

    SellerLive->>Source: Mo ban flash sale san pham Y
    Source->>Pipeline: Nhung BROADCAST_EVENT vao luong video (khong doi checkout)
    Pipeline-->>Pipeline: Tiep tuc phan phoi on dinh cho toan bo viewer, khong bi anh huong

    par Lan song viewer bam mua gan nhu dong thoi
        Note over Queue: Toan bo yeu cau bam mua duoc day vao Queue rieng,<br/>khong goi truc tiep dong bo vao Checkout
        Queue->>Checkout: Phan phoi dan CHECKOUT_REQUEST theo kha nang xu ly (rate limit/backpressure)
        Checkout->>Stock: Doi chieu ton kho thuc cho tung request
        Checkout-->>Queue: Cap nhat ket qua (confirmed/rejected)
    end

    Note over Pipeline,Queue: Queue va Checkout chay tren ha tang tach biet, tu auto-scale rieng,<br/>tai tang dot bien o day khong chiem tai nguyen cua Pipeline video
```
