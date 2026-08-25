# Distributed tracing/observability pipeline flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (SaaS B2B nhiều microservice, e-commerce, fintech, mobile backend gọi third-party, pipeline xử lý media) để luyện việc theo dõi một request đi qua nhiều service và tổng hợp lại thành một câu chuyện có thể debug được.

---

## Truy vết request xuyên nhiều service trong nền tảng SaaS B2B

**Repository:** `distributed-tracing-b2b-saas-request`

**Hệ thống:** Một nền tảng SaaS quản lý dự án, một request tạo task đi qua service `api-gateway` → `task-service` → `notification-service` → `search-indexer` (bất đồng bộ).

**Vai trò của flow:** Distributed tracing dùng để nối các bước xử lý rải rác trên nhiều service thành một trace duy nhất, giúp debug khi một request "tạo task" bị chậm hoặc lỗi mà không rõ nghẽn ở đâu.

**Yêu cầu cụ thể:**
- Mỗi request phải được gắn một `trace_id` sinh ở gateway (nếu chưa có từ client) và propagate qua toàn bộ header của các lời gọi nội bộ tiếp theo, bao gồm cả lời gọi bất đồng bộ qua message queue.
- Mỗi service khi xử lý phải tạo một `span` con gắn với `trace_id` gốc và `parent_span_id` đúng, để dựng lại được cây gọi (call tree) đúng thứ tự và đúng quan hệ cha-con, kể cả khi các service gọi song song nhau.
- Xử lý được trường hợp một service không hỗ trợ tracing (legacy service bên thứ ba) — vẫn phải giữ được `trace_id` liên tục qua service đó (pass-through header) dù không có span chi tiết bên trong.
- Đảm bảo dữ liệu trace không làm rò rỉ thông tin nhạy cảm (không được gắn giá trị field chứa password/token vào tên span hay tag) — có allowlist rõ ràng cho các attribute được phép ghi vào trace.
- Tracing pipeline phải chịu được việc một service gửi trace data trễ hoặc mất gói (network drop) mà không làm sai lệch cây trace của các request khác — mỗi span phải tự đứng độc lập được, không phụ thuộc thứ tự nhận.

---

## Theo dõi hành trình một đơn hàng qua các service của e-commerce

**Repository:** `distributed-tracing-ecommerce-order-journey`

**Hệ thống:** Một sàn e-commerce có luồng đặt hàng đi qua `order-service`, `inventory-service`, `payment-service`, `shipping-service`, mỗi service log riêng và có thể scale độc lập.

**Vai trò của flow:** Khi một khách hàng báo "đơn hàng của tôi bị treo", team vận hành cần dùng trace để xác định chính xác bước nào trong 4 service trên đang chậm hoặc lỗi, không cần grep log từng service một cách thủ công.

**Yêu cầu cụ thể:**
- Mỗi span phải ghi lại đủ thông tin để phân biệt lỗi do logic nghiệp vụ (ví dụ hết hàng) với lỗi hạ tầng (timeout, connection refused) — hai loại này cần cách xử lý cảnh báo khác nhau.
- Đo và ghi lại latency riêng biệt cho từng bước, đồng thời tổng hợp được latency toàn trình (end-to-end) của một order, để phân biệt "chậm ở một bước" với "chậm tích lũy đều ở mọi bước".
- Xử lý được các bước xử lý bất đồng bộ (ví dụ payment callback tới sau vài phút qua webhook) mà vẫn liên kết đúng vào trace ban đầu của đơn hàng đó, không tạo thành trace rời rạc riêng.
- Định nghĩa cơ chế sampling: không nhất thiết trace 100% request (chi phí lưu trữ lớn), nhưng phải đảm bảo 100% các trace có lỗi (error) hoặc latency vượt ngưỡng được giữ lại đầy đủ (tail-based sampling), không bị bộ lọc sampling ngẫu nhiên bỏ qua.
- Cung cấp khả năng tra cứu nhanh: từ `order_id` của khách hàng, tìm ra đúng `trace_id` liên quan (cần lưu mapping này) để hỗ trợ khi khách hàng khiếu nại mà không biết trace_id kỹ thuật.

---

## Observability cho pipeline xử lý giao dịch fintech cần audit chặt

**Repository:** `distributed-tracing-fintech-audit-observability`

**Hệ thống:** Một hệ thống xử lý chuyển tiền giữa các ví điện tử, mỗi giao dịch phải đi qua kiểm tra fraud, ghi ledger, và gửi thông báo — mọi bước phải có thể truy vết lại cho mục đích compliance.

**Vai trò của flow:** Tracing không chỉ dùng để debug performance mà còn đóng vai trò audit trail kỹ thuật — chứng minh được thứ tự và thời điểm chính xác của từng bước xử lý một giao dịch.

**Yêu cầu cụ thể:**
- Trace data liên quan tới giao dịch tài chính phải được lưu trữ với thời hạn (retention) dài hơn trace thông thường (theo yêu cầu compliance), và không được tự động xóa theo policy sampling/rotation mặc định của hệ thống observability.
- Đảm bảo tính toàn vẹn: span ghi lại thời điểm và kết quả của bước kiểm tra fraud không được sửa đổi được sau khi ghi (append-only), để tránh tình huống tranh chấp về việc "hệ thống đã check fraud hay chưa".
- Xử lý trường hợp một bước trong pipeline gọi ra một service bên thứ ba (ngân hàng đối tác) có độ trễ cao và không kiểm soát được — span phải ghi rõ đây là external call, tách biệt để không tính nhầm vào SLA nội bộ.
- Thiết kế cảnh báo tự động khi một giao dịch có trace "dừng bất thường" giữa chừng (ví dụ span "ledger-write" không có bước tiếp theo trong X giây) — dấu hiệu có thể là giao dịch bị treo cần can thiệp thủ công.
- Đảm bảo việc thu thập trace không làm tăng đáng kể latency của chính giao dịch (tracing overhead phải được đo và giữ dưới một ngưỡng phần trăm cụ thể so với latency gốc).

---

## Mobile backend gọi nhiều API bên thứ ba (thanh toán, bản đồ, SMS)

**Repository:** `distributed-tracing-mobile-backend-third-party-apis`

**Hệ thống:** Backend cho một app mobile đặt xe, mỗi request đặt xe phải gọi tới cổng thanh toán, dịch vụ bản đồ, và dịch vụ gửi SMS xác nhận — đều là service bên thứ ba.

**Vai trò của flow:** Vì phần lớn độ trễ và lỗi thực tế nằm ở các lời gọi ra ngoài, pipeline tracing phải làm rõ được service bên thứ ba nào đang là điểm nghẽn hoặc gây lỗi cho trải nghiệm người dùng.

**Yêu cầu cụ thể:**
- Mỗi lời gọi ra third-party API phải được bọc trong một span riêng ghi rõ tên provider, endpoint, response code, và latency — để có thể tổng hợp thống kê theo từng provider (ví dụ "provider bản đồ X chậm hơn Y 3 lần vào giờ cao điểm").
- Xử lý retry: khi một lời gọi third-party được retry (do timeout), mỗi lần retry phải là một span riêng nhưng liên kết vào cùng một "logical operation" span cha, để không hiểu nhầm 3 lần retry là 3 hành động độc lập.
- Trace phải phân biệt được lỗi do timeout phía client (app đợi không đủ lâu) với lỗi do provider trả về chậm/hỏng, để đội ngũ biết nên điều chỉnh timeout hay đổi provider.
- Đảm bảo dữ liệu trace không lưu lại thông tin thẻ thanh toán hoặc số điện thoại đầy đủ (phải mask/hash) dù các trường này xuất hiện trong request/response tới third-party.
- Thiết kế dashboard (hoặc mô tả yêu cầu cho dashboard) tổng hợp theo trace để trả lời nhanh câu hỏi "trong 1000 request đặt xe gần nhất, bao nhiêu % thời gian tiêu tốn ở bên thứ ba so với logic nội bộ".

---

## Pipeline xử lý media (transcode video) nhiều bước bất đồng bộ

**Repository:** `distributed-tracing-video-transcode-pipeline`

**Hệ thống:** Một dịch vụ cho phép người dùng upload video, sau đó video được xử lý qua nhiều bước bất đồng bộ: kiểm duyệt nội dung, transcode nhiều độ phân giải, tạo thumbnail, rồi publish.

**Vai trò của flow:** Vì các bước chạy trên nhiều worker khác nhau và có thể mất từ vài giây tới vài phút, tracing giúp biết một video đang ở bước nào và bước nào đang là nút cổ chai chung của toàn hệ thống.

**Yêu cầu cụ thể:**
- Toàn bộ các job xử lý (kiểm duyệt, transcode từng độ phân giải, tạo thumbnail) của một video phải liên kết vào cùng một trace gốc dù chạy trên các worker pool khác nhau và có thể chạy song song.
- Với các bước chạy song song (ví dụ transcode 3 độ phân giải cùng lúc), trace phải thể hiện đúng quan hệ song song (sibling span), không xếp chúng thành tuần tự sai lệch thời gian thực tế.
- Khi một bước bị lỗi và job được retry tự động sau một khoảng thời gian (backoff), span của lần retry phải ghi rõ thời gian chờ (đã cố ý delay) tách biệt với thời gian xử lý thật, để không tính sai vào "thời gian xử lý trung bình".
- Cung cấp khả năng tổng hợp theo từng bước trên toàn hệ thống (không chỉ theo một video) để trả lời "bước transcode 1080p đang chậm dần theo thời gian, có phải do tài nguyên worker đang thiếu".
- Xử lý trường hợp một video bị người dùng xóa giữa lúc đang xử lý — các span phát sinh sau đó (ví dụ job vẫn đang chạy trong queue) phải được đóng lại rõ ràng với trạng thái "cancelled", không để trace treo ở trạng thái "đang xử lý" mãi mãi trên dashboard giám sát.
