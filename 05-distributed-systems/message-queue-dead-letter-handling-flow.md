# Message queue dead-letter handling flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần xử lý message xử lý thất bại — queue đơn hàng e-commerce, queue gửi email/notification, queue xử lý ảnh/video upload, queue đối soát thanh toán, và queue lệnh điều khiển IoT — nhằm luyện cách phân loại lỗi tạm thời/vĩnh viễn, cách ly message lỗi, và cung cấp công cụ requeue/điều tra trong web app thực tế.

---

## Queue xử lý đơn hàng khi message xử lý liên tục lỗi

**Repository:** `dead-letter-order-processing-queue`

**Hệ thống:** E-commerce dùng message queue để xử lý các bước sau khi tạo đơn (trừ kho, tính phí, gửi email), một số message xử lý lỗi do dữ liệu bất thường hoặc bug.

**Vai trò của flow:** Dead-letter queue (DLQ) bắt các message xử lý thất bại quá số lần retry cho phép, tách ra để không làm nghẽn queue chính và cho phép xử lý/điều tra riêng.

**Yêu cầu cụ thể:**
- Định nghĩa rõ số lần retry tối đa cho mỗi message trước khi chuyển vào DLQ (ví dụ 5 lần với backoff), và message trong DLQ không được tự động retry lại vào queue chính.
- Message chuyển vào DLQ phải giữ nguyên toàn bộ payload gốc kèm metadata lỗi (lý do lỗi lần cuối, số lần đã retry, timestamp) để phục vụ điều tra không cần đoán.
- Có alert tự động khi DLQ có message mới hoặc khi số lượng message trong DLQ vượt ngưỡng, để team vận hành biết ngay có vấn đề đang xảy ra ở pipeline xử lý đơn hàng.
- Cung cấp công cụ (dashboard/CLI) cho phép xem, sửa payload (nếu cần) và requeue lại message từ DLQ vào queue chính sau khi đã fix nguyên nhân lỗi.
- Đảm bảo việc một message bị lỗi liên tục và vào DLQ không làm chặn (block) các message khác phía sau nó trong queue chính (message lỗi không được giữ queue, phải bị skip sau khi hết retry).

---

## Queue gửi email/notification khi provider bên ngoài lỗi

**Repository:** `dead-letter-notification-provider-failure`

**Hệ thống:** Service gửi email/notification cho user thông qua queue, đôi khi provider email bên ngoài trả lỗi vĩnh viễn (địa chỉ không hợp lệ) hoặc lỗi tạm thời (rate limit).

**Vai trò của flow:** Phân biệt lỗi tạm thời (nên retry) và lỗi vĩnh viễn (nên vào DLQ ngay, retry không giải quyết được gì), tối ưu để không lãng phí tài nguyên retry vô ích.

**Yêu cầu cụ thể:**
- Phải phân loại lỗi rõ ràng: lỗi 4xx do email không hợp lệ/không tồn tại đưa thẳng vào DLQ không retry, lỗi 5xx/timeout/429 thì retry với backoff trước khi vào DLQ.
- Message trong DLQ do email không hợp lệ phải kích hoạt một luồng riêng (ví dụ đánh dấu địa chỉ email đó là "invalid" trong hệ thống user, tránh gửi tiếp các lần sau).
- Có cơ chế gộp báo cáo định kỳ (ví dụ hàng ngày) tổng hợp message trong DLQ theo loại lỗi để team phát hiện xu hướng (ví dụ một provider bắt đầu block domain email công ty).
- Đảm bảo throughput xử lý message chính không bị ảnh hưởng khi có lượng lớn message cùng lúc rơi vào lỗi và cần route tới DLQ (routing tới DLQ phải nhẹ, không tốn nhiều tài nguyên như xử lý chính).
- Cho phép cấu hình TTL cho message nằm trong DLQ (ví dụ tự xóa sau 30 ngày nếu không ai xử lý) để tránh DLQ phình vô hạn theo thời gian.

---

## Queue xử lý ảnh/video upload khi file lỗi hoặc worker crash

**Repository:** `dead-letter-media-upload-worker-crash`

**Hệ thống:** Nền tảng chia sẻ media dùng queue để xử lý encode/resize ảnh, video sau khi user upload, một số file bị lỗi format hoặc quá lớn khiến worker xử lý thất bại/crash.

**Vai trò của flow:** DLQ cách ly các job xử lý file lỗi ra khỏi luồng chính, tránh một file lỗi làm crash liên tục worker và làm nghẽn toàn bộ hàng đợi xử lý các file bình thường khác.

**Yêu cầu cụ thể:**
- Nếu một job làm worker crash (không phải lỗi trả về bình thường mà là process die), hệ thống phải phát hiện qua visibility timeout hết hạn và đưa job đó vào DLQ sau một số lần crash lặp lại, không để nó tiếp tục "giết" các worker khác lần lượt.
- File lỗi format/corrupt phải được nhận diện nhanh (validate trước khi xử lý nặng) để tránh tốn tài nguyên xử lý rồi mới fail — job dạng này nên fail-fast vào DLQ ngay từ vài giây đầu.
- Có công cụ cho team support xem chi tiết lý do một file/job cụ thể rơi vào DLQ (ví dụ để trả lời user tại sao video của họ "xử lý mãi không xong").
- DLQ phải lưu được đủ thông tin để tái tạo job (ví dụ đường dẫn file gốc trên storage) để có thể requeue xử lý lại sau khi worker được fix hoặc file được validate lại thủ công.
- Đo lường: tỉ lệ job rơi vào DLQ theo loại file/định dạng để phát hiện pattern lỗi (ví dụ một codec cụ thể luôn gây lỗi) và ưu tiên fix worker theo dữ liệu thực tế.

---

## Queue đối soát giao dịch thanh toán (payment reconciliation)

**Repository:** `dead-letter-payment-reconciliation-queue`

**Hệ thống:** Hệ thống tài chính xử lý message đối soát giao dịch giữa nội bộ và cổng thanh toán, một số message không khớp được do dữ liệu lệch hoặc race condition.

**Vai trò của flow:** DLQ giữ lại các message đối soát không xử lý được để con người review thủ công, vì đây là dữ liệu tài chính nhạy cảm không thể tự động bỏ qua hay retry mù.

**Yêu cầu cụ thể:**
- Message đối soát lỗi tuyệt đối không được tự động retry vô hạn hay tự động "giả định đúng" — phải vào DLQ kèm đầy đủ chi tiết sai lệch (số tiền kỳ vọng vs thực tế, transaction ID liên quan) để người review.
- Có SLA rõ ràng về thời gian tối đa một message tài chính được nằm trong DLQ trước khi phải có người xử lý (ví dụ trong vòng giờ làm việc tiếp theo), kèm escalation nếu quá hạn.
- Sau khi người vận hành xử lý xong (đã điều chỉnh dữ liệu/xác nhận), phải có hành động rõ ràng: requeue để hệ thống tự xử lý lại, hoặc đánh dấu "đã xử lý thủ công" và đóng case, không để lửng lơ.
- Log toàn bộ lịch sử một message đối soát: vào DLQ khi nào, ai xử lý, hành động gì, kết quả cuối — phục vụ audit tài chính.
- Có phân loại mức độ nghiêm trọng (ví dụ chênh lệch số tiền lớn ưu tiên xử lý trước chênh lệch nhỏ do làm tròn) để đội vận hành xử lý theo đúng thứ tự ưu tiên rủi ro.

---

## Queue gửi lệnh điều khiển tới thiết bị IoT

**Repository:** `dead-letter-iot-command-queue`

**Hệ thống:** Nền tảng IoT gửi lệnh điều khiển (mở/đóng, cập nhật cấu hình) tới thiết bị qua message queue, một số thiết bị offline hoặc phản hồi lỗi liên tục.

**Vai trò của flow:** DLQ giữ các lệnh gửi thất bại nhiều lần (thiết bị offline lâu) để không giữ chúng mãi trong queue chính làm nghẽn lệnh gửi cho thiết bị khác đang online.

**Yêu cầu cụ thể:**
- Lệnh gửi tới thiết bị offline phải retry với backoff hợp lý (không dồn quá nhanh vì thiết bị IoT thường băng thông thấp), và sau ngưỡng thời gian/số lần nhất định phải chuyển vào DLQ.
- Khi thiết bị kết nối lại sau thời gian dài offline, hệ thống phải quyết định rõ: có tự động gửi lại lệnh cũ trong DLQ hay không (một số lệnh có thể đã lỗi thời, ví dụ lệnh "tắt đèn lúc 8h tối" gửi lại lúc 10h sáng hôm sau không còn ý nghĩa).
- Mỗi lệnh trong DLQ phải có TTL nghiệp vụ rõ ràng (không phải TTL kỹ thuật chung) để tự động loại bỏ lệnh đã hết ý nghĩa thời gian, tránh gửi lệnh cũ gây hành vi sai cho thiết bị.
- Có cơ chế nhóm (group) các lệnh DLQ theo thiết bị để khi thiết bị online lại, hệ thống xử lý theo đúng thứ tự nghiệp vụ hợp lý (ví dụ chỉ giữ lệnh cấu hình mới nhất, bỏ các lệnh cấu hình cũ hơn đã bị lệnh mới override).
- Giám sát tỉ lệ thiết bị có lệnh tồn đọng trong DLQ theo khu vực/loại thiết bị để phát hiện vấn đề hạ tầng mạng theo vùng.
