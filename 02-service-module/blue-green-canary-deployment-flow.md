# Blue-green/Canary deployment flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (SaaS B2B multi-tenant, e-commerce checkout, mobile backend API versioning, hệ thống core banking, hạ tầng phục vụ mô hình machine learning) để luyện việc triển khai phiên bản mới an toàn, có thể rollback nhanh, và giảm thiểu rủi ro ảnh hưởng người dùng thật.

---

## Blue-green deployment cho service lõi của nền tảng SaaS multi-tenant

**Repository:** `blue-green-deployment-saas-core-service`

**Hệ thống:** Một nền tảng SaaS B2B phục vụ nhiều tenant (công ty khách hàng) dùng chung hạ tầng, service "core-api" được deploy nhiều lần mỗi tuần.

**Vai trò của flow:** Blue-green deployment cho phép chuyển toàn bộ traffic từ phiên bản cũ (blue) sang phiên bản mới (green) gần như ngay lập tức, và rollback tức thì nếu phát hiện sự cố sau khi chuyển.

**Yêu cầu cụ thể:**
- Trước khi chuyển traffic, phiên bản green phải được deploy đầy đủ song song với blue (không tắt blue), và trải qua bước smoke test tự động (gọi các endpoint quan trọng) xác nhận hoạt động đúng trước khi router bắt đầu chuyển traffic thật sang.
- Việc chuyển traffic từ blue sang green phải là một thao tác gần như tức thời ở tầng router/load balancer (đổi target), không yêu cầu client phải làm gì, và phải có khả năng đảo ngược (chuyển lại về blue) trong vài giây nếu phát hiện lỗi ngay sau khi chuyển.
- Xử lý đúng vấn đề tương thích dữ liệu khi có migration schema DB đi kèm bản deploy mới — phiên bản blue (cũ) vẫn đang chạy phải tiếp tục hoạt động đúng với schema đã thay đổi cho tới khi bị tắt hẳn (yêu cầu migration phải backward-compatible trong giai đoạn chuyển tiếp).
- Định nghĩa rõ tiêu chí tự động rollback (ví dụ tỷ lệ lỗi 5xx vượt ngưỡng X% trong Y phút đầu sau khi chuyển) và cơ chế giám sát phải chạy ngay sau khi chuyển traffic, không chờ phát hiện thủ công từ báo cáo người dùng.
- Đảm bảo các session/kết nối đang mở tại thời điểm chuyển đổi (ví dụ WebSocket, request dài) được xử lý đúng — không bị cắt ngang giữa lúc router đổi target, mà phải hoàn tất trên blue hoặc được chuyển tiếp một cách có kiểm soát.

---

## Triển khai thay đổi cho hệ thống core banking cần kiểm soát chặt và có thể audit

**Repository:** `deployment-core-banking-audit-controlled`

**Hệ thống:** Một hệ thống core banking xử lý giao dịch tài khoản, chịu quy định chặt về thay đổi hệ thống (change management, audit), rất hạn chế rủi ro downtime hoặc lỗi dữ liệu.

**Vai trò của flow:** Canary/blue-green ở đây phải đi kèm với quy trình phê duyệt, khả năng audit đầy đủ, và mức độ thận trọng cao hơn nhiều so với các hệ thống web thông thường.

**Yêu cầu cụ thể:**
- Mọi giai đoạn của canary rollout (bắt đầu, tăng tỷ lệ, rollback) phải được ghi lại đầy đủ trong audit log không thể sửa đổi (ai thực hiện, khi nào, tỷ lệ traffic tại mỗi thời điểm), phục vụ yêu cầu kiểm toán và tuân thủ quy định ngành ngân hàng.
- Giai đoạn canary ban đầu chỉ nên áp dụng cho các giao dịch có giá trị thấp hoặc rủi ro thấp (ví dụ chỉ giao dịch xem số dư, chưa cho giao dịch chuyển tiền) trước khi mở rộng dần sang các loại giao dịch quan trọng hơn, thay vì chia đều theo tỷ lệ traffic ngẫu nhiên như hệ thống thông thường.
- Đảm bảo mọi thay đổi liên quan tới việc tính toán số dư/giao dịch phải có khả năng đối soát song song giữa phiên bản cũ và mới trong giai đoạn canary (chạy shadow — gửi cùng request tới cả hai phiên bản, so sánh kết quả, nhưng chỉ trả về kết quả từ phiên bản đang chính thức) để phát hiện sai lệch trước khi thực sự chuyển traffic.
- Định nghĩa quy trình rollback không chỉ về mã nguồn mà cả về dữ liệu — nếu phiên bản mới đã ghi dữ liệu theo định dạng mới trước khi phát hiện lỗi cần rollback, phải có kế hoạch xử lý dữ liệu đó khi quay lại phiên bản cũ, tránh mất tính nhất quán số liệu tài chính.
- Yêu cầu bước phê duyệt thủ công (không tự động hoàn toàn) trước khi chuyển từ mỗi giai đoạn canary sang giai đoạn tiếp theo, có ít nhất hai người xác nhận (four-eyes principle) trước khi mở rộng lên 100% traffic.

---

## Canary deployment cho việc thay thế mô hình machine learning đang serving

**Repository:** `canary-deployment-ml-model-replacement`

**Hệ thống:** Một hệ thống gợi ý sản phẩm (recommendation) dùng mô hình machine learning để serving kết quả real-time cho hàng triệu request mỗi ngày, cần thay mô hình mới định kỳ.

**Vai trò của flow:** Khác với deploy code thông thường, ở đây "phiên bản mới" là một mô hình đã train, cần đánh giá dựa trên chất lượng kết quả gợi ý (business metric) không chỉ đúng/sai kỹ thuật.

**Yêu cầu cụ thể:**
- Mô hình mới phải được canary trên một phần nhỏ traffic thật, và kết quả gợi ý phải được đo bằng các metric nghiệp vụ (tỷ lệ click, tỷ lệ chuyển đổi mua hàng) so sánh với mô hình cũ đang chạy song song trên phần traffic còn lại, không chỉ đo latency/lỗi kỹ thuật.
- Xử lý trường hợp mô hình mới hoạt động kỹ thuật hoàn toàn ổn định (không lỗi, latency tốt) nhưng chất lượng gợi ý kém hơn mô hình cũ về mặt nghiệp vụ — vẫn phải coi đây là tiêu chí rollback, không chỉ dựa vào lỗi hệ thống.
- Đảm bảo việc phân chia traffic vào canary của mô hình mới không làm sai lệch dữ liệu dùng để train các mô hình tương lai (tránh feedback loop — mô hình mới ảnh hưởng hành vi user rồi dữ liệu đó lại dùng để train tiếp gây thiên lệch không kiểm soát).
- Thiết kế thời gian canary đủ dài để loại trừ yếu tố thời điểm (ví dụ hành vi mua hàng khác nhau giữa ngày thường và cuối tuần) trước khi kết luận mô hình mới tốt hơn hay kém hơn, tránh quyết định dựa trên mẫu dữ liệu quá ngắn/thiên lệch.
- Có khả năng rollback tức thời về mô hình cũ nếu phát hiện mô hình mới serving ra kết quả bất thường/không phù hợp (ví dụ gợi ý sản phẩm không liên quan hoặc vi phạm chính sách nội dung) mà không cần chờ phân tích đầy đủ metric dài hạn.
