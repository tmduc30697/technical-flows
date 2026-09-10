# Sequence Diagram — Enhance: Adaptive Heartbeat Over Backup Link

Đây là **enhance**, flow hoàn toàn mới cho thiết bị đang chạy trên đường truyền dự phòng (SIM 4G giới hạn dung lượng) — báo cáo trạng thái theo chu kỳ dài hơn để tiết kiệm dữ liệu, kèm cơ chế nền tảng chủ động gửi lệnh kiểm tra khẩn qua kênh riêng khi nghi ngờ sự cố, thay vì chờ chu kỳ báo cáo tiếp theo.

```mermaid
sequenceDiagram
    participant Device as Thiết bị POS
    participant Platform as Nền tảng quản lý POS
    actor Ops as Đội vận hành

    Device->>Device: Phát hiện mất cáp, chuyển sang SIM 4G dự phòng
    Device->>Platform: Heartbeat (link_type=backup), báo đổi chu kỳ báo cáo dài hơn
    Platform->>Platform: Update DEVICE current_link_type=backup

    loop chu kỳ dài hơn qua backup link
        Device->>Platform: Heartbeat tiết kiệm dữ liệu (link_type=backup)
    end

    Platform->>Platform: Phát hiện im lặng bất thường ngay cả với chu kỳ dài đã điều chỉnh
    Platform->>Device: Gửi lệnh kiểm tra khẩn qua kênh riêng, không chờ chu kỳ báo cáo tiếp theo
    alt thiết bị phản hồi
        Device-->>Platform: Xác nhận vẫn hoạt động, link_type=backup
        Platform->>Platform: Giữ status=active
    else không phản hồi lệnh khẩn
        Platform->>Platform: Mark DEVICE status=disconnected_suspect
        Platform->>Ops: Cảnh báo cần xử lý
    end
```
