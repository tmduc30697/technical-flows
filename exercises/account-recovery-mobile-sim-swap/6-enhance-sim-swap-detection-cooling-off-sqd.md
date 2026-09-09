# Enhance sequence — SIM-swap detection & cooling-off

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu 2 và 4 của đề bài: phát hiện tín hiệu đổi SIM (từ nhà mạng hoặc thiết bị lạ hoàn toàn so với lịch sử), áp cooling-off, nhưng không chặn cứng mà kết hợp thêm tín hiệu khác (lịch sử thiết bị, backup email) để phân biệt người dùng hợp lệ với kẻ tấn công.

```mermaid
sequenceDiagram
    participant Carrier as Nhà mạng (nếu có tín hiệu)
    participant App as Mobile App
    participant Monitor as SIM-swap Monitor
    participant DB as DEVICE / SIM_SWAP_EVENT store

    alt Có tín hiệu trực tiếp từ nhà mạng
        Carrier-->>Monitor: Báo hiệu số điện thoại vừa đổi SIM
    else Suy luận gián tiếp
        App-->>Monitor: OTP vừa được xác minh thành công trên 1 DEVICE hoàn toàn khác lịch sử (fingerprint không khớp bất kỳ device đã biết)
    end
    Monitor->>DB: Tạo SIM_SWAP_EVENT (signal_source, detected_at)
    Monitor->>DB: Đối chiếu thêm tín hiệu: lịch sử thiết bị, có backup_email đã verify khớp không, tốc độ thay đổi bất thường
    Monitor->>Monitor: Tính risk_level dựa trên tổ hợp tín hiệu (không chỉ dựa 1 tín hiệu duy nhất)
    alt risk_level = high (nghi vấn SIM-swap thật)
        Monitor->>DB: cooling_off_until = now + N giờ, status=active
        Note over DB: Trong lúc cooling-off, các thay đổi nhạy cảm từ số điện thoại này bị chặn (xem flow "Change sensitive info")
    else risk_level = low (dấu hiệu người dùng hợp lệ đổi máy/SIM chính chủ)
        Monitor->>DB: status=reviewed_benign, không áp cooling-off
        Note over DB: Người dùng tiếp tục dùng bình thường, không bị khóa nhầm
    end
```
