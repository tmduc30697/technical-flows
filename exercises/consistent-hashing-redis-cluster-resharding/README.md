# Resharding cụm cache phân tán (Redis Cluster-like) khi scale

**Hệ thống:** Một cụm cache dùng chung cho nhiều service, ban đầu có N node, cần scale thêm/giảm node theo tải mà không làm cache-miss toàn bộ.

**Vai trò của flow:** Consistent hashing quyết định key nào thuộc node nào, và khi thêm/bớt node chỉ một phần nhỏ key phải di chuyển (thay vì rehash toàn bộ).

**Yêu cầu cụ thể:**
- Khi thêm 1 node vào cụm N node, chỉ khoảng 1/(N+1) tổng số key phải di chuyển sang node mới — đo và chứng minh được tỉ lệ này bằng test thực tế.
- Dùng virtual node (nhiều điểm hash ảo cho mỗi node vật lý) để tránh phân bổ tải lệch (hot node) khi số node vật lý ít.
- Trong lúc resharding, request đọc/ghi vào key đang được di chuyển phải được xử lý đúng (không mất update, không đọc dữ liệu cũ) — nêu rõ chiến lược (dual-write tạm thời, hoặc route theo bảng ánh xạ đang chuyển tiếp).
- Có khả năng rollback resharding nếu phát hiện lỗi giữa quá trình migrate mà không làm mất dữ liệu đã migrate một phần.
- Đo lường: thời gian hoàn thành resharding cho một lượng dữ liệu cụ thể (ví dụ 100GB) và ảnh hưởng tới p99 latency của cache trong lúc resharding diễn ra.
