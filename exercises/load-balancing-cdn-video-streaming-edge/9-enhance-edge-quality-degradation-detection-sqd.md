# Enhance sequence — Edge quality degradation detection

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ dựa vào health check hệ thống cơ bản, không có khái niệm chất lượng dịch vụ thực tế. Đáp ứng **yêu cầu 5** (phát hiện edge node suy giảm chất lượng dù vẫn "healthy" theo health check cơ bản, ví dụ tỉ lệ rebuffer tăng cao, và giảm dần trọng số route trước khi tình trạng nghiêm trọng hơn).

```mermaid
sequenceDiagram
    participant Edge as Edge Node (health check vẫn pass)
    participant Player as Video Player (client)
    participant QualityMonitor as Quality Monitor
    participant Router as Edge Router

    loop Liên tục trong lúc phát
        Player->>QualityMonitor: Báo cáo REBUFFER_SAMPLE (rebuffer_ratio của viewer này)
    end
    Edge-->>QualityMonitor: Health check hệ thống cơ bản vẫn trả về "healthy"

    QualityMonitor->>QualityMonitor: Tổng hợp rebuffer_ratio trung bình của node trong cửa sổ gần nhất
    alt Rebuffer ratio trung bình vượt ngưỡng dù health check vẫn pass
        QualityMonitor->>Edge: Cập nhật quality_score giảm xuống
        QualityMonitor->>Router: Yêu cầu giảm dần routing_weight của node này
        Router->>Router: Giảm dần tỉ lệ route viewer mới tới node, không cắt hẳn ngay lập tức
        Note over Router,Edge: Viewer mới dần được route sang node khác trước khi tình trạng rebuffer trở nên nghiêm trọng hơn
    else Rebuffer ratio bình thường
        QualityMonitor->>Router: Giữ nguyên routing_weight
    end
```
