# Đặt combo du lịch (vé máy bay + khách sạn + thuê xe)

**Hệ thống:** Nền tảng đặt tour cho phép khách đặt đồng thời vé máy bay, khách sạn, xe thuê từ 3 nhà cung cấp/service khác nhau.

**Vai trò của flow:** Saga đảm bảo tính "tất cả hoặc không có gì" ở mức nghiệp vụ (không có 2-phase commit thật) — nếu một phần đặt thất bại, phải hủy các phần đã đặt thành công trước đó.

**Yêu cầu cụ thể:**
- Thứ tự đặt phải được thiết kế để phần dễ hủy/ít rủi ro nhất được đặt trước (ví dụ giữ chỗ tạm thời trước khi charge tiền thật).
- Mỗi service cung cấp (airline, hotel, car) có API riêng với SLA/latency khác nhau — saga phải có timeout riêng cho từng bước và coi timeout là "thất bại cần compensate".
- Compensate hủy khách sạn/vé máy bay có thể mất phí hủy (cancellation fee) — phải log và tính đúng số tiền hoàn lại cho khách, không hoàn nhầm 100% khi có phí hủy.
- Cho khách xem được trạng thái real-time của từng phần đặt (đang xử lý/thành công/đã hủy) trong lúc saga đang chạy.
- Có cơ chế phát hiện saga "kẹt" quá lâu ở một bước (worker xử lý bước đó chết) và tự động resume hoặc alert vận hành.
