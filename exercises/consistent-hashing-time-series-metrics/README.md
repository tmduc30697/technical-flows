# Sharding lưu trữ time-series metrics theo thời gian và theo nguồn

**Hệ thống:** Một hệ thống giám sát (monitoring) nhận metrics từ hàng chục nghìn service, cần lưu time-series data với throughput ghi rất cao.

**Vai trò của flow:** Partition dữ liệu theo cả thời gian (time-based partition) và theo nguồn (hash theo metric/service) để ghi/đọc song song hiệu quả, tránh một partition trở thành nghẽn cổ chai.

**Yêu cầu cụ thể:**
- Phải chọn chiến lược partition kết hợp (composite key: hash(service) + time bucket) và giải thích lý do tránh được "hot partition" khi một service đẩy metrics dồn dập.
- Truy vấn theo khoảng thời gian (ví dụ metrics 1 giờ qua của 1 service) phải chỉ chạm vào số partition tối thiểu cần thiết, không phải scan toàn bộ cụm.
- Dữ liệu partition cũ (quá X ngày) phải tự động được nén/archive sang storage rẻ hơn mà không làm gián đoạn ghi dữ liệu mới.
- Đảm bảo throughput ghi (write) tối thiểu (ví dụ 1 triệu điểm dữ liệu/giây toàn cụm) và nêu rõ số node/partition cần để đạt mức đó.
- Có cơ chế phát hiện partition bị lệch tải nghiêm trọng (một service quá lớn so với các service khác) và tách riêng partition dành cho nó (partition splitting theo nhu cầu).
