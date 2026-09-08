# Điều phối deploy đa vùng (multi-region) tránh deploy trùng

**Hệ thống:** Một nền tảng SaaS chạy ở 3 region, có pipeline CI/CD tự động deploy, cần đảm bảo không có 2 region tự deploy đồng thời gây trạng thái không đồng nhất.

**Vai trò của flow:** Dùng distributed lock toàn cục (cross-region) để chỉ một pipeline deploy được chạy tại một thời điểm, các pipeline khác phải chờ tới lượt.

**Yêu cầu cụ thể:**
- Lock phải được đặt ở một coordinator có khả năng chịu lỗi cao (ví dụ ZooKeeper ensemble hoặc etcd cluster) không phụ thuộc vào bất kỳ region đang deploy.
- Nếu pipeline giữ lock bị treo (do lỗi CI) quá thời gian tối đa cho phép, lock phải tự nhả và có cảnh báo cho người vận hành, không chặn deploy khác vô thời hạn.
- Phải log đầy đủ audit: region nào deploy, lúc nào, ai trigger, để phục vụ rollback/điều tra sự cố.
- Xử lý kịch bản coordinator mất kết nối tạm thời với 1 region: region đó không được tự cho là "an toàn để deploy" nếu không xác nhận được lock hợp lệ.
- Đưa ra được giới hạn thời gian chờ tối đa cho pipeline đang chờ lock (queue timeout) và hành vi khi vượt quá (hủy job, thông báo, retry).
