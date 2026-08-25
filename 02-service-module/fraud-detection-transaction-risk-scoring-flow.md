# Fraud detection & transaction risk scoring flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (e-commerce, fintech chuyển tiền, marketplace, ride-hailing, dịch vụ streaming) để luyện việc chấm điểm rủi ro và chặn/giữ giao dịch nghi ngờ mà không làm phiền quá mức người dùng hợp lệ.

---

## Chấm điểm rủi ro cho giao dịch thanh toán trên sàn e-commerce

**Repository:** `fraud-detection-ecommerce-payment-risk-score`

**Hệ thống:** Một sàn e-commerce xử lý thanh toán bằng thẻ, cần chặn giao dịch gian lận (thẻ ăn cắp, chargeback) trước khi xác nhận đơn hàng.

**Vai trò của flow:** Mỗi giao dịch thanh toán được chấm điểm rủi ro dựa trên nhiều tín hiệu (địa chỉ IP, lịch sử mua hàng, giá trị đơn) để quyết định approve tự động, yêu cầu xác minh thêm, hay chặn.

**Yêu cầu cụ thể:**
- Thiết kế pipeline chấm điểm chạy được trong thời gian thực (dưới vài trăm ms) để không làm chậm trải nghiệm checkout, kể cả khi cần gọi thêm dịch vụ chấm điểm bên thứ ba.
- Định nghĩa rõ 3 mức hành động theo ngưỡng điểm: approve tự động, yêu cầu xác minh bổ sung (OTP/3D Secure), và chặn thẳng — kèm log lại lý do (feature nào góp phần chính vào điểm số) để phục vụ khiếu nại sau này.
- Xử lý race condition: cùng một thẻ thực hiện nhiều giao dịch gần như đồng thời ở nhiều tab/thiết bị (dấu hiệu gian lận phổ biến) — hệ thống phải phát hiện được dựa trên tốc độ giao dịch (velocity check), không chỉ chấm từng giao dịch độc lập.
- Phải có cơ chế phản hồi ngược (feedback loop): khi một giao dịch bị đánh false positive (khách hàng hợp lệ bị chặn oan) và được xác minh thủ công là hợp lệ, thông tin này phải được ghi lại để cải thiện mô hình chấm điểm, không lặp lại lỗi tương tự với khách hàng đó.
- Đảm bảo hệ thống vẫn hoạt động (fail-safe) khi dịch vụ chấm điểm rủi ro bên ngoài bị timeout/lỗi — định nghĩa rõ nên fail open (cho qua, chấp nhận rủi ro) hay fail closed (chặn, chấp nhận mất doanh thu) theo giá trị đơn hàng.

---

## Phát hiện giao dịch chuyển tiền bất thường trong ứng dụng fintech

**Repository:** `fraud-detection-fintech-anomalous-transfer`

**Hệ thống:** Một ứng dụng ví điện tử cho phép chuyển tiền giữa người dùng, cần phát hiện các mẫu hình rửa tiền hoặc chiếm đoạt tài khoản (account takeover).

**Vai trò của flow:** Risk scoring đóng vai trò tuyến phòng thủ chính để giữ lại (hold) hoặc yêu cầu xác minh thêm cho các giao dịch có dấu hiệu bất thường trước khi tiền thực sự chuyển đi.

**Yêu cầu cụ thể:**
- Phát hiện thay đổi hành vi bất thường của một tài khoản (ví dụ đột nhiên chuyển số tiền lớn gấp nhiều lần trung bình lịch sử, hoặc chuyển tới một người nhận hoàn toàn mới ngay sau khi đổi mật khẩu/thiết bị) và tự động tăng mức độ xác minh cần thiết.
- Xử lý mẫu hình rửa tiền dạng "structuring" — chia nhỏ một số tiền lớn thành nhiều giao dịch nhỏ hơn ngưỡng báo cáo, thực hiện trong khoảng thời gian ngắn — cần chấm điểm tổng hợp theo cụm giao dịch (không chỉ theo từng giao dịch riêng lẻ).
- Khi một giao dịch bị giữ lại (hold) để xác minh thêm, phải có SLA rõ ràng về thời gian xử lý và cơ chế thông báo minh bạch cho người dùng, tránh giữ tiền vô thời hạn mà không giải thích.
- Đảm bảo tính nhất quán khi tính điểm rủi ro dựa trên số dư/lịch sử tài khoản trong điều kiện có nhiều giao dịch đồng thời đang chờ xử lý (không đọc số dư đã lỗi thời dẫn tới chấm điểm sai).
- Thiết kế cơ chế escalation: các giao dịch vượt một ngưỡng rủi ro cao nhất định phải được đẩy sang review thủ công của team compliance, kèm đầy đủ ngữ cảnh (lịch sử liên quan, điểm số breakdown) để con người ra quyết định nhanh mà không cần tra cứu lại từ đầu.

---

## Phát hiện gian lận trong onboarding người bán trên marketplace

**Repository:** `fraud-detection-marketplace-seller-onboarding`

**Hệ thống:** Một marketplace C2C cho phép bất kỳ ai đăng ký bán hàng, cần phát hiện các tài khoản người bán giả mạo hoặc hàng loạt (fake seller farm) trước khi cho phép nhận thanh toán.

**Vai trò của flow:** Risk scoring áp dụng ngay từ giai đoạn đăng ký và trong những giao dịch bán hàng đầu tiên, khi chưa có nhiều lịch sử để dựa vào.

**Yêu cầu cụ thể:**
- Chấm điểm rủi ro cho tài khoản mới phải dựa được vào các tín hiệu không cần lịch sử giao dịch (ví dụ thiết bị/IP trùng với các tài khoản đã bị khóa trước đó, tốc độ tạo tài khoản bất thường từ cùng một nguồn).
- Phát hiện mẫu hình "seller farm": nhiều tài khoản người bán khác nhau nhưng có cùng thông tin ngân hàng nhận tiền, cùng địa chỉ giao hàng, hoặc hành vi đăng sản phẩm giống nhau bất thường về thời gian.
- Xử lý trường hợp một người bán hợp lệ mới tham gia (không có lịch sử) bị chấm điểm rủi ro cao chỉ vì thiếu dữ liệu — cần phân biệt rõ "rủi ro cao do thiếu thông tin" với "rủi ro cao do có dấu hiệu xấu cụ thể", và áp dụng chính sách khác nhau (ví dụ giữ tiền trong thời gian ngắn ban đầu thay vì chặn thẳng).
- Đảm bảo hệ thống chấm điểm được cập nhật gần thời gian thực khi có báo cáo/khiếu nại từ người mua (return rate cao, khiếu nại hàng giả) để điều chỉnh điểm rủi ro của người bán đang hoạt động, không chỉ chấm một lần lúc đăng ký.
- Thiết kế để tránh việc người bán gian lận học được ngưỡng chặn (ví dụ cố tình đăng ký với thông tin gần đúng ngưỡng để lách) — không để logic ngưỡng cứng (hard threshold) dễ dự đoán bị khai thác.

---

## Chấm điểm rủi ro chuyến đi trong ứng dụng đặt xe

**Repository:** `fraud-detection-ride-hailing-trip-risk`

**Hệ thống:** Một ứng dụng đặt xe cần phát hiện gian lận giữa tài xế và khách (thông đồng tạo chuyến giả để nhận khuyến mãi, hoặc tài xế tự đặt chuyến cho chính mình).

**Vai trò của flow:** Risk scoring chạy sau mỗi chuyến đi hoàn tất để phát hiện các mẫu hình gian lận nhằm chiếm đoạt khuyến mãi hoặc gian lận doanh thu, phục vụ việc khóa tài khoản hoặc giữ lại thanh toán khuyến mãi.

**Yêu cầu cụ thể:**
- Phát hiện mẫu hình thông đồng: cùng một cặp tài khoản khách-tài xế lặp lại chuyến đi bất thường về tần suất hoặc tuyến đường (ví dụ chuyến rất ngắn nhưng lặp lại nhiều lần trong ngày chỉ để nhận khuyến mãi).
- Phát hiện tài xế tự đặt chuyến cho chính mình thông qua tài khoản khách phụ (dựa vào tín hiệu như vị trí GPS của hai thiết bị trùng nhau bất thường trong suốt chuyến đi).
- Xử lý được trường hợp dữ liệu GPS bị nhiễu hoặc giả lập (mock location) một cách có chủ đích để né tránh phát hiện — cần tín hiệu bổ sung ngoài GPS đơn thuần (ví dụ tốc độ di chuyển thực tế, cảm biến chuyển động của thiết bị).
- Đảm bảo việc giữ lại thanh toán khuyến mãi khi nghi ngờ gian lận không ảnh hưởng tới phần thanh toán chính đáng của chuyến đi (tài xế vẫn nhận được tiền cước hợp lệ, chỉ phần khuyến mãi nghi ngờ mới bị giữ).
- Thiết kế cơ chế cho phép người dùng bị khóa nhầm khiếu nại và được review lại nhanh, có audit log đầy đủ lý do hệ thống đưa ra quyết định (để tránh khóa oan gây mất người dùng thật).

---

## Phát hiện chia sẻ tài khoản gian lận trên dịch vụ streaming

**Repository:** `fraud-detection-streaming-account-sharing`

**Hệ thống:** Một dịch vụ streaming video theo mô hình subscription, giới hạn số thiết bị/địa điểm xem đồng thời trên một tài khoản.

**Vai trò của flow:** Risk scoring dùng để phát hiện các tài khoản bị chia sẻ vượt quá phạm vi cho phép (chia sẻ ngoài gia đình, bán lại tài khoản) để áp dụng chính sách nhắc nhở hoặc yêu cầu nâng cấp gói, khác với việc chặn gian lận thanh toán.

**Yêu cầu cụ thể:**
- Chấm điểm dựa trên các tín hiệu như số địa điểm địa lý khác nhau xem cùng lúc, khoảng cách địa lý bất hợp lý giữa các lần đăng nhập gần nhau (impossible travel), và số thiết bị/mạng khác nhau truy cập trong một khung thời gian ngắn.
- Phân biệt rõ giữa hành vi hợp lệ (gia đình đi công tác vẫn xem, dùng VPN hợp pháp) và hành vi vi phạm chính sách chia sẻ, tránh xử lý sai gây trải nghiệm xấu cho khách hàng trả tiền hợp lệ.
- Thiết kế chính sách phản ứng theo mức độ (soft — gửi cảnh báo/yêu cầu xác minh thêm; hard — buộc đăng xuất thiết bị cũ, giới hạn số stream đồng thời) thay vì khóa tài khoản ngay khi vượt ngưỡng lần đầu.
- Đảm bảo việc tính điểm không dựa vào một tín hiệu đơn lẻ dễ gây false positive (ví dụ chỉ dựa vào IP) mà phải tổng hợp nhiều tín hiệu theo thời gian trước khi đưa ra hành động ảnh hưởng tới trải nghiệm người dùng.
- Xử lý khả năng mở rộng: hệ thống phải chấm điểm được cho hàng triệu session xem đồng thời mà không làm chậm việc bắt đầu phát video (chấm điểm phải chạy bất đồng bộ/song song, không chặn luồng phát chính).
