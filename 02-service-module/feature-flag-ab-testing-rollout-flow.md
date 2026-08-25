# Feature flag/A-B testing rollout flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh web khác nhau (e-commerce checkout, mobile app, SaaS B2B pricing, dịch vụ streaming đề xuất nội dung, công cụ nội bộ doanh nghiệp) để luyện việc rollout tính năng có kiểm soát và chạy A/B test đúng phương pháp thống kê, không gây trải nghiệm rối loạn cho người dùng.

---

## A/B test thay đổi UI trang checkout của sàn e-commerce

**Repository:** `feature-flag-ecommerce-checkout-ab-test`

**Hệ thống:** Một sàn e-commerce muốn test một thiết kế mới cho trang checkout (bố cục nút thanh toán khác) để xem có làm tăng tỷ lệ hoàn tất đơn hàng không.

**Vai trò của flow:** Feature flag chia người dùng vào nhóm A (thiết kế cũ) và nhóm B (thiết kế mới) một cách ngẫu nhiên và nhất quán, để đo lường chính xác tác động của thay đổi UI lên hành vi mua hàng.

**Yêu cầu cụ thể:**
- Một người dùng phải luôn thấy cùng một phiên bản (A hoặc B) trong suốt vòng đời session/nhiều lần truy cập, không bị đổi qua đổi lại giữa hai phiên bản gây trải nghiệm không nhất quán và làm sai dữ liệu đo lường (dùng cơ chế bucket ổn định theo user_id/device_id, không dùng random mỗi lần load trang).
- Đảm bảo việc chia nhóm không tương quan với các yếu tố có thể gây thiên lệch kết quả (ví dụ không vô tình chia theo giờ trong ngày hoặc theo khu vực địa lý dẫn tới hai nhóm có đặc điểm khách hàng khác nhau một cách hệ thống).
- Định nghĩa rõ metric chính (tỷ lệ hoàn tất checkout) và metric phụ cần theo dõi (giá trị đơn hàng trung bình, tỷ lệ hủy giữa chừng) trước khi bắt đầu test, và tính cỡ mẫu cần thiết để kết quả có ý nghĩa thống kê trước khi kết luận, tránh dừng test sớm khi thấy số liệu tạm thời có vẻ khả quan (peeking bias).
- Xử lý trường hợp cần tắt gấp phiên bản B (feature flag kill switch) nếu phát hiện lỗi nghiêm trọng (ví dụ nút thanh toán bị lỗi không bấm được) — phải tắt được ngay lập tức không cần deploy lại code, và chuyển toàn bộ user đang ở nhóm B về lại nhóm A một cách êm, không làm mất thông tin giỏ hàng đang có.
- Sau khi test kết thúc và một phiên bản được chọn làm chính thức, phải có quy trình dọn dẹp feature flag khỏi codebase (không để flag cũ tồn tại vĩnh viễn gây rối logic), và migrate toàn bộ user về đúng phiên bản thắng.

---

## Rollout tính năng mới theo từng bước cho mobile app

**Repository:** `feature-flag-mobile-staged-rollout`

**Hệ thống:** Một mobile app muốn rollout một tính năng mới (ví dụ thanh toán bằng ví điện tử tích hợp) dần dần cho người dùng, thay vì bật cho toàn bộ user ngay khi phát hành bản cập nhật.

**Vai trò của flow:** Feature flag cho phép kiểm soát việc bật/tắt tính năng độc lập với việc phát hành bản app mới, giảm rủi ro khi tính năng có vấn đề chưa phát hiện qua testing.

**Yêu cầu cụ thể:**
- Việc bật/tắt tính năng phải điều khiển được từ xa (remote config) mà không cần người dùng cập nhật app mới, và app phải kiểm tra trạng thái flag ở thời điểm phù hợp (ví dụ khi mở app, hoặc định kỳ) không làm chậm thời gian khởi động app.
- Xử lý trường hợp app không có kết nối mạng khi cần kiểm tra flag (offline) — phải có giá trị mặc định an toàn (fail-safe, thường là tắt tính năng mới) và không được crash hoặc treo màn hình chờ phản hồi flag vô thời hạn.
- Rollout theo tỷ lệ tăng dần (ví dụ 5% → 20% → 50% → 100% người dùng) và phải theo dõi được crash rate/lỗi báo cáo riêng cho nhóm đã bật tính năng so với nhóm chưa bật, để phát hiện sớm nếu tính năng mới gây crash tăng vọt.
- Đảm bảo tính năng mới không được bật cho các version app quá cũ chưa được test với tính năng đó (flag phải kiểm tra cả điều kiện version app tối thiểu, không chỉ tỷ lệ phần trăm ngẫu nhiên).
- Cung cấp khả năng loại trừ/thêm cụ thể một người dùng vào nhóm test (ví dụ để nhân viên nội bộ hoặc người dùng beta luôn thấy tính năng mới trước) độc lập với cơ chế rollout theo tỷ lệ ngẫu nhiên chung.

---

## A/B test mô hình giá (pricing) mới cho SaaS B2B

**Repository:** `feature-flag-b2b-saas-pricing-ab-test`

**Hệ thống:** Một nền tảng SaaS B2B muốn test một cấu trúc giá subscription mới (đổi từ tính theo số seat sang tính theo mức sử dụng) với một nhóm khách hàng thử nghiệm trước khi áp dụng rộng.

**Vai trò của flow:** Vì đây là thay đổi ảnh hưởng trực tiếp tới hợp đồng và doanh thu, feature flag ở đây phải kiểm soát chặt hơn nhiều so với A/B test UI thông thường, và không thể chỉ dựa vào random đơn giản.

**Yêu cầu cụ thể:**
- Việc chọn khách hàng vào nhóm test giá mới phải qua một quy trình có kiểm soát rõ ràng (ví dụ chỉ khách hàng mới ký hợp đồng, có sự đồng ý rõ ràng), không được tự động áp giá mới cho khách hàng hiện tại mà không thông báo, vì đây ảnh hưởng tới hợp đồng đã ký.
- Đảm bảo hệ thống billing tính đúng và nhất quán theo đúng mô hình giá mà khách hàng đó thuộc về suốt cả kỳ hợp đồng, không bị lẫn giữa hai mô hình giá cũ/mới trong cùng một khách hàng do lỗi đọc flag không nhất quán giữa các lần tính hóa đơn.
- Xử lý trường hợp cần dừng test giữa chừng (ví dụ phát hiện mô hình giá mới có lỗi tính toán gây thu sai tiền khách hàng) — phải có khả năng chuyển khách hàng về lại mô hình giá cũ và tính toán lại đúng số tiền đã thu sai (hoàn tiền/thu bổ sung) một cách minh bạch, có thông báo rõ cho khách hàng.
- Định nghĩa metric đánh giá không chỉ về doanh thu ngắn hạn mà cả về tỷ lệ gia hạn hợp đồng (retention) của nhóm test so với nhóm đối chứng, vì thay đổi giá có thể ảnh hưởng hành vi khách hàng trong thời gian dài hơn một A/B test UI thông thường.
- Đảm bảo mọi thay đổi trạng thái flag liên quan tới giá của một khách hàng cụ thể được ghi log đầy đủ (ai thay đổi, khi nào, từ giá trị gì sang giá trị gì) phục vụ đối soát tài chính và giải quyết tranh chấp hợp đồng nếu khách hàng khiếu nại.

---

## Thử nghiệm thuật toán đề xuất nội dung mới trên dịch vụ streaming

**Repository:** `feature-flag-streaming-recommendation-ab-test`

**Hệ thống:** Một dịch vụ streaming video muốn test một thuật toán gợi ý nội dung mới trên trang chủ, so sánh với thuật toán hiện tại đang chạy.

**Vai trò của flow:** Feature flag chia người xem vào các nhóm thuật toán khác nhau và đo lường tác động lên hành vi xem (thời gian xem, tỷ lệ bỏ giữa chừng) — cần xử lý đặc thù dữ liệu hành vi thay đổi theo thời gian dài, không chỉ một hành động tức thời như checkout.

**Yêu cầu cụ thể:**
- Người xem phải được gán cố định vào một nhóm thuật toán trong suốt thời gian test (nhiều tuần) để đo được tác động dài hạn (ví dụ có tiếp tục subscription không), không bị đổi nhóm giữa các lần đăng nhập.
- Xử lý hiệu ứng "novelty" — người dùng có thể phản ứng tích cực với thuật toán mới chỉ vì nó mới lạ trong vài ngày đầu, cần thiết kế thời gian test đủ dài để phân biệt hiệu ứng thật với hiệu ứng tạm thời do sự mới lạ.
- Đảm bảo việc phân nhóm không tạo ra trải nghiệm bất công rõ rệt giữa hai nhóm theo cách người dùng có thể nhận ra và phàn nàn (ví dụ một thuật toán rõ ràng kém hơn nhiều dẫn tới trải nghiệm tệ hẳn) — cần giám sát chất lượng gợi ý theo thời gian thực và có ngưỡng dừng sớm nếu một nhóm có chỉ số trải nghiệm giảm bất thường.
- Với người dùng dùng nhiều thiết bị (TV, mobile, web) trên cùng một tài khoản, phải đảm bảo trải nghiệm thuật toán nhất quán trên mọi thiết bị của họ (cùng một tài khoản không bị chia vào hai nhóm khác nhau ở hai thiết bị khác nhau).
- Thiết kế khả năng phân tích kết quả theo từng phân khúc người dùng (ví dụ theo thể loại nội dung ưa thích) để phát hiện trường hợp thuật toán mới tốt hơn cho một số phân khúc nhưng kém hơn ở phân khúc khác, tránh kết luận quá đơn giản chỉ dựa vào số liệu tổng thể.

---

## Rollout công cụ nội bộ mới theo phòng ban trong doanh nghiệp

**Repository:** `feature-flag-internal-tool-department-rollout`

**Hệ thống:** Một công cụ quản lý quy trình làm việc nội bộ (workflow tool) được phát hành phiên bản mới, công ty muốn rollout dần theo từng phòng ban thay vì bật cho toàn bộ nhân viên ngay.

**Vai trò của flow:** Feature flag ở bối cảnh nội bộ doanh nghiệp không nhằm mục đích A/B test thống kê nghiêm ngặt như sản phẩm public, mà chủ yếu để quản lý rủi ro triển khai và thu thập phản hồi có kiểm soát.

**Yêu cầu cụ thể:**
- Việc bật tính năng mới phải cấu hình được theo nhóm tổ chức (phòng ban, team, hoặc danh sách email cụ thể) chứ không chỉ theo tỷ lệ phần trăm ngẫu nhiên, vì mục tiêu là kiểm soát ai được trải nghiệm trước theo quyết định của quản trị viên.
- Đảm bảo khi một nhân viên chuyển phòng ban hoặc một phòng ban được thêm/loại khỏi danh sách rollout, trạng thái tính năng của họ cập nhật đúng ngay ở lần truy cập tiếp theo, không cần họ đăng xuất/đăng nhập lại.
- Cung cấp cho quản trị viên hệ thống một giao diện quản lý flag đơn giản (không cần biết code) để họ tự thêm/xóa phòng ban khỏi danh sách rollout và xem được số lượng người dùng hiện đang thấy phiên bản mới.
- Xử lý trường hợp một số quy trình làm việc liên phòng ban (ví dụ một task được tạo bởi phòng ban đã có tính năng mới nhưng được xử lý tiếp bởi phòng ban chưa có) — phải đảm bảo dữ liệu/trạng thái công việc vẫn nhất quán và hiển thị đúng dù hai bên đang ở hai phiên bản UI khác nhau.
- Thu thập phản hồi có cấu trúc (không chỉ dựa vào feedback tự phát) từ các phòng ban đã được rollout trước khi mở rộng ra toàn công ty, và có tiêu chí rõ ràng (ví dụ số lượng report lỗi dưới một ngưỡng, phản hồi tích cực từ đa số) để quyết định tiếp tục mở rộng hay tạm dừng rollout.
