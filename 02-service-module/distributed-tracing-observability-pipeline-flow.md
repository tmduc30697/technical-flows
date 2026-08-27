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

---

## Truy vết một đơn hàng e-commerce qua giỏ hàng, tồn kho, thanh toán và xác nhận

**Repository:** `distributed-tracing-ecommerce-order-flow`

**Hệ thống:** Một sàn e-commerce có luồng đặt hàng đi qua bốn service tách biệt: `cart-service` → `inventory-service` (kiểm tra và giữ tồn kho) → `payment-service` (gọi cổng thanh toán bên ngoài) → `order-confirmation-service`, mỗi service có tài nguyên và tốc độ scale khác nhau.

**Vai trò của flow:** Khi checkout bị chậm vào giờ cao điểm, tracing phải chỉ ra chính xác bước nào đang là nút thắt (tồn kho đang bị lock tranh chấp, hay cổng thanh toán bên ngoài đang chậm, hay chỉ đơn giản là hàng đợi nội bộ đang nghẽn) để đội vận hành không phải đoán mò giữa nhiều nguyên nhân khả dĩ.

**Yêu cầu cụ thể:**
- Trace của một đơn hàng phải giữ nguyên `trace_id` xuyên suốt bốn bước dù chúng chạy trên các service độc lập với vòng đời request khác nhau (một số đồng bộ, một số phải chờ callback bất đồng bộ từ cổng thanh toán), để dựng lại được toàn bộ hành trình của một đơn hàng cụ thể khi khách hàng báo lỗi.
- Bước kiểm tra tồn kho thường liên quan tới việc chờ lock (nhiều đơn hàng cùng tranh chấp một sản phẩm hot), span của bước này phải tách riêng thời gian chờ lock khỏi thời gian xử lý logic thật, để phân biệt được "chậm vì tranh chấp tồn kho" với "chậm vì code xử lý nặng".
- Bước gọi cổng thanh toán bên ngoài phải được đánh dấu rõ là external dependency với SLA riêng, để khi tổng hợp thống kê "checkout chậm ở bước nào" vào giờ cao điểm, đội vận hành không nhầm lẫn giữa độ trễ do chính hệ thống gây ra và độ trễ nằm ngoài tầm kiểm soát của cổng thanh toán đối tác.
- Thiết kế dashboard tổng hợp theo từng bước trên toàn bộ traffic (không chỉ theo từng đơn hàng riêng lẻ) để trả lời câu hỏi "giờ cao điểm hôm nay chậm hơn hôm qua, chậm chủ yếu ở bước nào" — cho phép so sánh phân phối độ trễ (p50/p95/p99) từng bước theo thời gian thực.
- Xử lý trường hợp một đơn hàng bị timeout ở bước thanh toán nhưng cổng thanh toán thực ra đã xử lý thành công phía họ và gửi callback trễ — trace phải ghi lại đủ để đối chiếu, tránh tình huống hệ thống báo "đơn hàng thất bại" trong khi thực tế tiền đã bị trừ, gây khiếu nại khó điều tra nếu thiếu dữ liệu trace đầy đủ.

---

## Trace lời gọi từ mobile app xuyên qua backend tới các dịch vụ bên thứ ba

**Repository:** `distributed-tracing-mobile-backend-thirdparty`

**Hệ thống:** Một ứng dụng mobile gọi vào backend, và với một số tính năng, backend phải gọi tiếp ra các dịch vụ bên thứ ba (bản đồ/định vị, gửi SMS xác thực, đẩy push notification) trước khi trả kết quả về cho app.

**Vai trò của flow:** Tracing phải phân biệt rõ ràng thời gian chờ các dịch vụ bên thứ ba (ngoài tầm kiểm soát của đội kỹ thuật) với thời gian backend tự xử lý nội bộ, để khi app "cảm giác chậm", đội vận hành biết nên gây sức ép với đối tác thứ ba hay tối ưu code nội bộ.

**Yêu cầu cụ thể:**
- Mỗi lời gọi ra dịch vụ bên thứ ba phải được bọc trong một span riêng, gắn tag rõ ràng là external call kèm tên nhà cung cấp, tách biệt hoàn toàn khỏi span xử lý logic nội bộ, để khi tổng hợp latency trung bình của một API, có thể bóc tách được phần "chờ bên ngoài" ra khỏi phần "backend tự xử lý".
- Vì kết nối từ mobile app tới backend qua mạng di động không ổn định, phải phân biệt được độ trễ do chính mạng di động của client (trước khi request tới được backend) với độ trễ phát sinh trong nội bộ hệ thống backend — tránh gộp chung khiến số liệu latency nội bộ bị nhiễu bởi yếu tố phía client hoàn toàn không kiểm soát được.
- Xử lý trường hợp một dịch vụ bên thứ ba (ví dụ SMS) timeout hoặc trả lỗi và backend có cơ chế retry — mỗi lần retry phải tạo span riêng biệt kèm số thứ tự lần thử, để phân biệt được "một cuộc gọi chậm" với "ba lần gọi cộng dồn lại thành chậm", vốn cần xử lý khác nhau về mặt vận hành.
- Thiết kế cảnh báo riêng theo từng nhà cung cấp bên thứ ba (SLA latency/tỷ lệ lỗi riêng cho bản đồ, SMS, push notification) để phát hiện khi một đối tác cụ thể đang suy giảm chất lượng dịch vụ, thay vì chỉ có một cảnh báo chung chung "API chậm" gộp tất cả nguyên nhân lại.
- Đảm bảo dữ liệu trace không vô tình ghi lại nội dung nhạy cảm được gửi qua các dịch vụ bên thứ ba (ví dụ nội dung tin nhắn SMS chứa mã OTP, tọa độ vị trí chính xác của người dùng) vào tag/log của span — chỉ ghi metadata cần thiết cho việc debug hiệu năng (thời gian, trạng thái, mã lỗi), không ghi nội dung payload thật.
