# CAP theorem & tunable consistency (quorum read/write) flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần đánh đổi giữa availability và consistency — giỏ hàng e-commerce, follow/follower mạng xã hội, metrics giám sát hạ tầng, session checkout, và bộ đếm tồn kho flash sale — nhằm luyện cách cấu hình quorum (N, W, R), xử lý network partition theo đúng CAP, và công khai rõ mức consistency guarantee trong web app thực tế.

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

---

## Bộ đếm follower và quan hệ follow/unfollow cho mạng xã hội

**Repository:** `cap-theorem-social-follow-follower-counter`

**Hệ thống:** Mạng xã hội có tính năng follow/unfollow, và hiển thị số lượng follower trên profile cho hàng triệu user cùng lúc.

**Vai trò của flow:** Tunable consistency cho phép số đếm follower hiển thị nhanh (đọc gần đúng, eventual consistency chấp nhận sai lệch nhỏ tạm thời) trong khi thao tác follow/unfollow thực tế (quan hệ ai đang follow ai) phải chính xác để không gây lỗi logic nghiêm trọng hơn (như hiển thị nhầm quan hệ hoặc để user bấm follow lặp lại).

**Yêu cầu cụ thể:**
- Số đếm follower hiển thị trên profile được phép đọc từ counter tổng hợp bất đồng bộ (cập nhật dần theo batch hoặc CRDT counter), chấp nhận sai lệch tạm thời vài giây tới vài phút so với số thực tế, vì đây chỉ là con số hiển thị không ảnh hưởng logic nghiệp vụ khác.
- Ngược lại, hành động follow/unfollow (bản ghi quan hệ user A follow user B) phải ghi với write quorum đủ cao để đảm bảo ngay sau khi API follow trả thành công, mọi truy vấn tiếp theo (ví dụ kiểm tra "đã follow chưa" để hiển thị nút follow/following) đọc đúng trạng thái mới, tránh hiển thị sai nút bấm khiến user bấm follow nhiều lần.
- Race condition cụ thể: user bấm follow rồi unfollow rất nhanh liên tiếp (double-tap do UI lag) — 2 request này có thể tới 2 node khác nhau gần như đồng thời, cần cơ chế xác định thứ tự đúng (timestamp/version theo từng cặp quan hệ follow) để trạng thái cuối cùng phản ánh đúng ý định cuối của user, tránh "trạng thái không xác định" (đang follow nhưng số đếm coi như đã unfollow).
- Trong lúc network partition giữa các node lưu quan hệ follow, phải quyết định rõ: ưu tiên availability cho hành động follow (chấp nhận ghi ở phía minority, dung hòa/merge khi partition hàn lại theo kiểu "follow thắng unfollow nếu trùng thời điểm không xác định được") hay từ chối ghi nếu không đạt quorum — nêu rõ lựa chọn và hệ quả trải nghiệm user tương ứng.
- Đo lường: độ lệch giữa số đếm follower hiển thị (approximate counter) và số thực tế đếm trực tiếp từ bảng quan hệ (ground truth), chạy đối soát định kỳ và tự động điều chỉnh lại counter nếu độ lệch vượt ngưỡng cho phép.

---

## Metrics giám sát hạ tầng nội bộ cho dashboard tổng hợp và cảnh báo khẩn cấp

**Repository:** `cap-theorem-infra-metrics-dashboard-alerting`

**Hệ thống:** Nền tảng giám sát hạ tầng nội bộ (infrastructure monitoring) nhận metrics (CPU, memory, disk, latency...) liên tục từ hàng nghìn server/container trong nhiều cụm khác nhau, lưu trên cụm lưu trữ time-series phân tán để phục vụ dashboard tổng hợp và cảnh báo khẩn cấp khi tài nguyên gần cạn.

**Vai trò của flow:** Cho phép dữ liệu phục vụ dashboard tổng hợp/biểu đồ xu hướng chạy ở chế độ eventual consistency (đọc nhanh, chấp nhận trễ vài giây), trong khi luồng cảnh báo khi một server vượt ngưỡng nguy hiểm (ví dụ ổ đĩa sắp đầy, memory gần cạn) phải đọc dữ liệu mới nhất với độ chính xác cao hơn để tránh bỏ sót sự cố thực.

**Yêu cầu cụ thể:**
- Dashboard tổng hợp (biểu đồ trung bình/xu hướng theo thời gian của nhiều server) được phép đọc từ replica với R thấp, chấp nhận dữ liệu có thể trễ vài giây so với giá trị mới nhất, vì mục đích là quan sát xu hướng chứ không phải phản ứng tức thời.
- Luồng phát hiện ngưỡng nguy hiểm (ví dụ dung lượng đĩa vượt mức cảnh báo sắp đầy) phải đọc trực tiếp giá trị mới nhất với read quorum cao hơn hoặc đọc trực tiếp từ node vừa nhận ghi, vì đọc từ replica bị trễ có thể khiến hệ thống không kích hoạt cảnh báo kịp thời dù giá trị thực đã vượt ngưỡng từ trước đó, dẫn tới server sập mà không ai được báo trước.
- Trong lúc network partition giữa các node lưu trữ, agent thu thập metrics trên từng server vẫn tiếp tục gửi dữ liệu (không dừng lại chờ mạng ổn định) — cần quyết định rõ dữ liệu ghi vào phía minority có được chấp nhận tạm thời (ưu tiên availability, không mất dữ liệu giám sát) hay bị từ chối, và nêu rõ cách xử lý khi 2 phía có dữ liệu ghi chồng lấn thời điểm lúc partition hàn lại (ví dụ giữ theo timestamp agent gửi, không theo thời điểm node nhận).
- Xử lý trường hợp một agent gửi dữ liệu chậm/trễ do mất kết nối tạm thời rồi gửi bù (backfill) dữ liệu cũ — dashboard tổng hợp không nên tính sai lệch vì dữ liệu tới muộn, nhưng nếu giá trị backfill đó thực ra đã vượt ngưỡng nguy hiểm tại thời điểm xảy ra, cần có cơ chế đánh giá lại cảnh báo hồi tố (retroactive alert) thay vì bỏ qua vì đã "quá muộn".
- Đo lường: độ trễ giữa thời điểm server thực sự chạm ngưỡng nguy hiểm tới lúc hệ thống cảnh báo thực sự kích hoạt (end-to-end alert latency), và tỉ lệ đọc dashboard bị stale vượt ngưỡng chấp nhận được, tách riêng theo từng luồng (dashboard vs alert) để đánh giá đúng trade-off đang áp dụng.
