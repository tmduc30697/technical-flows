# CAP theorem & tunable consistency (quorum read/write) flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần đánh đổi giữa availability và consistency — giỏ hàng e-commerce, follow/follower mạng xã hội, dữ liệu cảm biến IoT, session checkout, và bộ đếm tồn kho flash sale — nhằm luyện cách cấu hình quorum (N, W, R), xử lý network partition theo đúng CAP, và công khai rõ mức consistency guarantee trong web app thực tế.

---

## Distributed key-value store cho giỏ hàng (shopping cart)

**Repository:** `cap-theorem-shopping-cart-kv-store`

**Hệ thống:** E-commerce lưu giỏ hàng của user trên một key-value store phân tán (kiểu DynamoDB/Cassandra) nhiều replica.

**Vai trò của flow:** Cho phép tunable consistency — cấu hình số replica cần xác nhận khi đọc/ghi (quorum) để cân bằng giữa tính khả dụng và tính nhất quán của giỏ hàng.

**Yêu cầu cụ thể:**
- Định nghĩa cụ thể tham số N (số replica), W (write quorum), R (read quorum) cho giỏ hàng, và giải thích rõ tại sao chọn W+R > N (đảm bảo đọc thấy ghi gần nhất) hay không.
- Trong lúc network partition giữa các replica, hệ thống phải ưu tiên availability cho hành động "thêm vào giỏ" (chấp nhận ghi ở phía minority, merge sau) vì trải nghiệm mua hàng quan trọng hơn nhất quán tuyệt đối tức thời.
- Khi partition được hàn lại, phải có chiến lược merge rõ ràng nếu 2 phía cùng sửa giỏ hàng khác nhau (ví dụ union các item được thêm, không tự động xóa item của bên nào).
- Cho phép client chọn mức consistency theo tình huống (ví dụ đọc giỏ hàng lúc checkout phải dùng read quorum cao hơn đọc để hiển thị icon số lượng ở header).
- Đo lường và log rõ tần suất phải "resolve conflict" xảy ra trong thực tế để đánh giá mức độ ảnh hưởng của việc chọn AP thay vì CP.

---

## Session store cần nhất quán cao hơn cho hành động nhạy cảm (thanh toán)

**Repository:** `cap-theorem-payment-session-consistency`

**Hệ thống:** Một platform thương mại điện tử lưu session/trạng thái checkout trên store phân tán, đa số truy cập chỉ đọc nhưng bước thanh toán cần chính xác tuyệt đối.

**Vai trò của flow:** Cho phép "tunable consistency" theo API cụ thể — phần lớn request dùng eventual consistency cho tốc độ, riêng bước xác nhận thanh toán chuyển sang strong consistency (CP) để tránh double-charge hay thấy trạng thái sai.

**Yêu cầu cụ thể:**
- Xác định rõ danh sách API nào chạy ở chế độ AP (đọc nhanh, có thể stale) và API nào buộc phải CP (ví dụ xác nhận đã thanh toán, khóa order để tránh 2 request cùng xử lý).
- Với API CP, khi xảy ra network partition và không đủ quorum, hệ thống phải từ chối rõ ràng (trả lỗi "tạm không xử lý được") thay vì âm thầm trả kết quả sai/không chắc chắn.
- Với API AP, phải có cơ chế phát hiện và cảnh báo khi độ "stale" vượt ngưỡng bất thường (ví dụ do một node bị treo lâu) để vận hành can thiệp.
- Test cụ thể case: 2 request thanh toán trùng gửi tới 2 node khác nhau trong lúc network chập chờn — phải đảm bảo chỉ một request thực sự được xác nhận thành công.
- Viết rõ tài liệu cho team khác (API consumer) biết mức consistency guarantee của từng endpoint để họ dùng đúng cách, tránh giả định sai.

---

## Bộ đếm tồn kho theo mô hình quorum-based cho flash sale

**Repository:** `cap-theorem-flash-sale-quorum-counter`

**Hệ thống:** E-commerce chạy flash sale với số lượng giới hạn, hệ thống tồn kho phân tán trên nhiều node để chịu tải đọc/ghi khổng lồ trong thời gian ngắn.

**Vai trò của flow:** Dùng cấu hình quorum (W, R) linh hoạt theo giai đoạn: trước flash sale ưu tiên tốc độ đọc, trong lúc flash sale ưu tiên chính xác ghi để tránh oversell.

**Yêu cầu cụ thể:**
- Trước giờ flash sale, đọc tồn kho hiển thị cho user có thể dùng R thấp (nhanh, cache-friendly) vì số lượng còn nhiều và sai lệch nhỏ không ảnh hưởng.
- Ngay khi tồn kho giảm dưới ngưỡng nguy hiểm (ví dụ còn <5% so với ban đầu), hệ thống phải tự chuyển API trừ tồn kho sang chế độ quorum ghi cao hơn (W lớn hơn) để giảm rủi ro oversell, kể cả phải chấp nhận latency tăng.
- Trong lúc network partition giữa các node giữ tồn kho, phần minority phải từ chối trừ kho (không tự ý bán) để tránh vượt tồn kho thực khi partition hàn lại.
- Có cơ chế đối soát cuối flash sale: tổng số bán ra theo ghi nhận hệ thống phải khớp với tồn kho vật lý thực tế, và có báo cáo chênh lệch nếu có.
- Benchmark rõ số request/giây hệ thống chịu được ở từng mode (R thấp trước sale, W cao lúc gần hết hàng) để lập kế hoạch capacity.
