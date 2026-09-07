# Bộ đếm tồn kho theo mô hình quorum-based cho flash sale

**Hệ thống:** E-commerce chạy flash sale với số lượng giới hạn, hệ thống tồn kho phân tán trên nhiều node để chịu tải đọc/ghi khổng lồ trong thời gian ngắn.

**Vai trò của flow:** Dùng cấu hình quorum (W, R) linh hoạt theo giai đoạn: trước flash sale ưu tiên tốc độ đọc, trong lúc flash sale ưu tiên chính xác ghi để tránh oversell.

**Yêu cầu cụ thể:**
- Trước giờ flash sale, đọc tồn kho hiển thị cho user có thể dùng R thấp (nhanh, cache-friendly) vì số lượng còn nhiều và sai lệch nhỏ không ảnh hưởng.
- Ngay khi tồn kho giảm dưới ngưỡng nguy hiểm (ví dụ còn <5% so với ban đầu), hệ thống phải tự chuyển API trừ tồn kho sang chế độ quorum ghi cao hơn (W lớn hơn) để giảm rủi ro oversell, kể cả phải chấp nhận latency tăng.
- Trong lúc network partition giữa các node giữ tồn kho, phần minority phải từ chối trừ kho (không tự ý bán) để tránh vượt tồn kho thực khi partition hàn lại.
- Có cơ chế đối soát cuối flash sale: tổng số bán ra theo ghi nhận hệ thống phải khớp với tồn kho vật lý thực tế, và có báo cáo chênh lệch nếu có.
- Benchmark rõ số request/giây hệ thống chịu được ở từng mode (R thấp trước sale, W cao lúc gần hết hàng) để lập kế hoạch capacity.
