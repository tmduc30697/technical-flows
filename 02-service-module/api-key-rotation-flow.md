# API key rotation flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (SaaS B2B nền tảng API, microservices nội bộ, cổng thanh toán cho merchant, webhook bên thứ ba, mobile backend) để luyện việc xoay vòng API key/secret mà không làm gián đoạn dịch vụ đang chạy.

---

## Xoay vòng API key cho khách hàng dùng nền tảng SaaS API

**Repository:** `api-key-rotation-saas-customer`

**Hệ thống:** Một nền tảng SaaS cung cấp API cho khách hàng doanh nghiệp tích hợp (gửi SMS, xác minh danh tính), mỗi khách hàng có một hoặc nhiều API key để gọi API.

**Vai trò của flow:** Cho phép khách hàng tự xoay vòng API key định kỳ hoặc khi nghi ngờ bị lộ, mà không làm gián đoạn hệ thống production đang tích hợp của họ.

**Yêu cầu cụ thể:**
- Cho phép một khách hàng có nhiều API key hoạt động đồng thời (không giới hạn 1 key/tài khoản), để họ có thể tạo key mới, cập nhật vào hệ thống của mình, rồi mới vô hiệu hóa key cũ (zero-downtime rotation).
- Mỗi API key phải lưu được metadata: thời điểm tạo, lần dùng gần nhất, scope/quyền hạn, và nhãn (label) do khách hàng tự đặt để dễ nhận biết key nào dùng cho môi trường nào (staging/production).
- Khi một key bị vô hiệu hóa (revoke), request dùng key đó phải bị chặn ngay ở tầng gateway/authentication, có độ trễ lan truyền tối thiểu (không cache quyền hợp lệ của key quá lâu ở các service khác nhau gây tình trạng key đã revoke vẫn dùng được vài phút).
- Cảnh báo tự động cho khách hàng khi một key không được dùng trong thời gian dài (có thể là key bị bỏ quên, nên revoke) và khi một key có dấu hiệu bị lộ (gọi từ nhiều địa chỉ IP/địa lý bất thường trong thời gian ngắn).
- Đảm bảo giá trị thật của API key không bao giờ được hiển thị lại sau lần tạo đầu tiên (chỉ hiển thị một lần), và chỉ lưu dạng hash trong hệ thống, tương tự cách lưu mật khẩu.

---

## Xoay vòng secret dùng cho giao tiếp service-to-service nội bộ

**Repository:** `api-key-rotation-internal-service-to-service`

**Hệ thống:** Một nền tảng gồm nhiều microservice nội bộ giao tiếp với nhau qua API, xác thực bằng shared secret/service token cấu hình tĩnh trong từng service.

**Vai trò của flow:** Định kỳ xoay vòng các secret này theo chính sách bảo mật nội bộ (ví dụ mỗi 90 ngày) mà không gây lỗi giao tiếp giữa các service trong lúc chuyển đổi.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế cho phép một service chấp nhận đồng thời secret cũ và secret mới trong một khoảng thời gian chuyển tiếp (dual-key validation), để các service gọi tới có thời gian cập nhật secret mới mà không bị gián đoạn.
- Định nghĩa rõ quy trình rotation: secret mới phải được phân phối tới tất cả service liên quan trước khi secret cũ bị vô hiệu hóa, và có bước xác nhận (health check/canary call) rằng mọi service đã cập nhật thành công trước khi tắt secret cũ.
- Xử lý trường hợp một service bị bỏ sót trong quá trình cập nhật (do config drift, deploy lỗi) — phải có cảnh báo rõ ràng khi secret cũ vẫn còn được dùng bởi ai đó sau thời điểm dự kiến vô hiệu hóa, tránh tắt secret một cách mù quáng theo lịch mà làm sập giao tiếp thật.
- Secret không được lưu ở dạng plaintext trong file config commit vào source control hay trong log — phải dùng secret manager (vault-like) và service đọc secret tại runtime, hỗ trợ reload secret mới mà không cần restart toàn service.
- Ghi log đầy đủ ai/khi nào thực hiện rotation, và có khả năng rollback nhanh về secret cũ trong trường hợp secret mới gây lỗi lan rộng phát hiện sau khi đã rotate.

---

## Xoay vòng API key merchant cho cổng thanh toán

**Repository:** `api-key-rotation-payment-merchant`

**Hệ thống:** Một cổng thanh toán cung cấp API key cho các merchant (cửa hàng online) để gọi API tạo giao dịch thanh toán, mỗi key gắn với quyền hạn tài chính thực tế.

**Vai trò của flow:** Vì key ở đây liên quan trực tiếp tới tiền, rotation phải đi kèm kiểm soát chặt hơn API key thông thường, đặc biệt khi nghi ngờ key bị lộ giữa lúc giao dịch đang chạy.

**Yêu cầu cụ thể:**
- Phân biệt rõ hai loại key: key công khai (publishable, dùng ở client-side, ít rủi ro hơn) và key bí mật (secret, dùng ở server-side merchant, cần bảo vệ cao) — chính sách rotation và mức độ cảnh báo khi lộ phải khác nhau giữa hai loại.
- Khi merchant báo cáo nghi ngờ secret key bị lộ, hệ thống phải hỗ trợ vô hiệu hóa khẩn cấp (emergency revoke) trong vài giây, đồng thời tự động rà soát các giao dịch gần nhất dùng key đó để phát hiện giao dịch bất thường phát sinh trong lúc key bị lộ.
- Đảm bảo trong lúc merchant đang rotate key (có cả key cũ và mới cùng hoạt động), hệ thống nội bộ ghi log rõ giao dịch nào dùng key nào, để nếu phát hiện gian lận có thể xác định chính xác key nào là nguồn rò rỉ.
- Giới hạn quyền hạn tài chính có thể gắn vào một key (ví dụ giới hạn số tiền tối đa/giao dịch, giới hạn loại thao tác được phép — chỉ tạo charge, không được refund) để giảm thiệt hại tối đa nếu một key bị lộ.
- Thiết kế cảnh báo tự động khi một key thực hiện khối lượng giao dịch tăng vọt bất thường so với lịch sử của merchant đó, xem đây là dấu hiệu cần xác minh dù chưa chắc key đã lộ.

---

## Xoay vòng secret ký webhook cho hệ thống tích hợp với bên thứ ba

**Repository:** `api-key-rotation-webhook-signing-secret`

**Hệ thống:** Một nền tảng e-commerce nhận webhook từ nhiều đối tác bên ngoài (cổng vận chuyển, cổng thanh toán) và cũng gửi webhook ra cho merchant của mình, mỗi webhook được ký bằng secret riêng để xác thực nguồn gốc.

**Vai trò của flow:** Rotation secret ký webhook phải đảm bảo cả hai chiều (nhận và gửi) không bị "điếc" giữa lúc secret thay đổi, vì webhook là giao tiếp bất đồng bộ và bên nhận có thể không online đúng lúc đổi secret.

**Yêu cầu cụ thể:**
- Với webhook đến (nhận từ đối tác), hệ thống phải chấp nhận verify chữ ký bằng cả secret cũ và secret mới trong một cửa sổ thời gian chuyển tiếp, vì đối tác có thể cập nhật secret không đồng thời với mình.
- Với webhook đi (gửi cho merchant), phải cung cấp cho merchant API/portal để họ tự lấy secret mới và có thời gian cập nhật hệ thống của họ trước khi secret cũ bị vô hiệu hóa, tránh việc đột ngột đổi secret khiến merchant không verify được webhook và mất dữ liệu quan trọng (như thông báo đơn hàng).
- Xử lý trường hợp webhook bị gửi thất bại trong lúc secret đang chuyển đổi — cơ chế retry (có backoff) của webhook phải dùng đúng secret tại thời điểm gửi lại, không dùng nhầm secret đã bị thu hồi.
- Đảm bảo mỗi request webhook mang theo timestamp và chữ ký bao gồm cả timestamp đó, để chống replay attack (một webhook cũ bị chặn và gửi lại sau) độc lập với việc rotation secret.
- Cung cấp log lịch sử rotation secret cho từng đối tác/merchant (khi nào đổi, secret cũ hết hiệu lực khi nào) để hỗ trợ debug khi có báo cáo "không nhận được webhook" sau một lần đổi secret.

---

## Xoay vòng API key nhúng trong mobile app

**Repository:** `api-key-rotation-embedded-mobile`

**Hệ thống:** Một mobile app gọi tới backend bằng một API key nhúng sẵn trong app (dùng để nhận diện app hợp lệ, không phải định danh user), key này phân phối cùng bản build app.

**Vai trò của flow:** Rotation ở đây khó hơn các trường hợp khác vì không thể ép mọi user update app ngay lập tức — phải xử lý được việc nhiều version app với nhiều key khác nhau cùng tồn tại ngoài thực tế trong thời gian dài.

**Yêu cầu cụ thể:**
- Backend phải chấp nhận đồng thời nhiều API key hợp lệ tương ứng với nhiều version app đang được cài trên thiết bị người dùng thực tế, không chỉ hai key (cũ/mới) như rotation thông thường.
- Có cơ chế theo dõi được tỷ lệ request theo từng key/version, để biết khi nào đủ an toàn (đủ ít user còn dùng version cũ) mới thực sự vô hiệu hóa một key cũ mà không ảnh hưởng số lượng lớn user.
- Với các version app quá cũ dùng key đã bị vô hiệu hóa, backend phải trả về lỗi rõ ràng yêu cầu cập nhật app (kèm mã lỗi riêng để app hiển thị đúng thông báo), không trả lỗi chung khiến user không hiểu vì sao app không dùng được.
- Đảm bảo API key nhúng trong app (dù có thể bị decompile lấy ra) không được gắn quyền hạn nhạy cảm — chỉ dùng để nhận diện nguồn gọi (app hợp lệ), còn định danh và quyền hạn thực sự của user phải dựa vào access token riêng lấy được sau khi user đăng nhập.
- Thiết kế chính sách rotation định kỳ theo chu kỳ phát hành app (aligned với release cycle) và có kế hoạch truyền thông/force-update cho các trường hợp bắt buộc phải rotate sớm (ví dụ key bị lộ công khai) dù một số user chưa cập nhật app.
