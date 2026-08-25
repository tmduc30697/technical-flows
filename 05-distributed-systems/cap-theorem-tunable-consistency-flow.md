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

## Kho lưu follow/follower cho mạng xã hội

**Repository:** `cap-theorem-social-follow-graph`

**Hệ thống:** Mạng xã hội lưu quan hệ follow/follower trên store phân tán, chịu tải đọc rất cao (ai follow ai) và ghi tương đối (follow/unfollow).

**Vai trò của flow:** Tối ưu cho đọc nhanh và khả dụng cao (thiên về AP), chấp nhận có thể thấy trạng thái follow "cũ" trong thời gian ngắn thay vì chặn user khi mạng chập chờn.

**Yêu cầu cụ thể:**
- Ghi follow/unfollow phải thành công ngay cả khi một phần replica không phản hồi kịp (write vào node local trước, replicate nền), không được để user chờ hoặc lỗi vì 1 node chậm.
- Đọc số lượng follower có thể là "gần đúng" (eventually consistent) và hệ thống phải công khai (qua doc/API) mức độ trễ tối đa cho phép (ví dụ vài giây).
- Với hành động nhạy cảm hơn (ví dụ chặn/block user), phải chuyển sang mức consistency cao hơn (đọc/ghi quorum chặt) để tránh trường hợp user vẫn thấy nội dung của người đã block do dữ liệu chưa đồng bộ.
- Xử lý conflict khi follow rồi unfollow rất nhanh liên tiếp trong lúc một phần hệ thống bị partition — trạng thái cuối cùng phải phản ánh đúng hành động gần nhất của user theo thời gian thực (not lost update).
- Có công cụ đo được "staleness" thực tế (khoảng trễ giữa hành động và khi mọi replica đồng bộ) trong vận hành thật.

---

## Kho dữ liệu cảm biến IoT chấp nhận mất một phần dữ liệu để giữ khả dụng

**Repository:** `cap-theorem-iot-sensor-availability`

**Hệ thống:** Nền tảng IoT thu thập dữ liệu cảm biến (nhiệt độ, độ ẩm) từ hàng chục nghìn thiết bị, lưu vào store phân tán nhiều vùng.

**Vai trò của flow:** Ưu tiên availability để luôn nhận được dữ liệu ghi vào từ thiết bị (không được từ chối ghi dù một phần cluster mất kết nối), đánh đổi có thể tạm thời không nhất quán giữa các bản đọc.

**Yêu cầu cụ thể:**
- Thiết bị gửi dữ liệu phải luôn ghi được vào node gần nhất available, không phụ thuộc vào việc các node khác có đang phân vùng mạng hay không.
- Đọc dữ liệu cho dashboard giám sát real-time chấp nhận đọc từ 1 replica (R=1) để tối ưu latency, nhưng phải hiển thị rõ "dữ liệu tính đến thời điểm X" để người xem biết độ trễ.
- Đối với truy vấn phân tích lịch sử (không real-time), phải chuyển sang đọc với quorum cao hơn để đảm bảo không thiếu dữ liệu do đọc từ replica chưa kịp đồng bộ.
- Xử lý trường hợp một vùng bị mất kết nối dài (nhiều giờ): dữ liệu thu thập trong lúc đó vẫn phải toàn vẹn ở vùng local và tự động merge/replicate khi kết nối khôi phục, không bị đè mất bởi dữ liệu ở vùng khác.
- Đo lường và báo cáo được tỉ lệ dữ liệu bị trễ đồng bộ trên 1 phút, trên 1 giờ, để đánh giá SLA thực tế của hệ thống.

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
