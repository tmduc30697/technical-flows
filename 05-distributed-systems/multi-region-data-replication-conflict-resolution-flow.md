# Multi-region data replication & conflict resolution flow — Đề bài thực hành

Các đề bài dưới đây đi qua nhiều bối cảnh cần đồng bộ dữ liệu đa vùng — user profile SaaS toàn cầu, editor cộng tác thời gian thực, tồn kho e-commerce đa vùng, counter mạng xã hội, và thứ tự tin nhắn chat — nhằm luyện thiết kế consistency model, chiến lược giải quyết xung đột, và xử lý network partition kéo dài trong web app thực tế.

---

## User profile store active-active nhiều vùng cho SaaS toàn cầu

**Repository:** `multi-region-replication-saas-user-profile`

**Hệ thống:** SaaS phục vụ khách hàng toàn cầu, dữ liệu profile người dùng được ghi ở nhiều region (US, EU, APAC) để giảm latency, với replication hai chiều giữa các vùng.

**Vai trò của flow:** Đảm bảo user ở bất kỳ vùng nào cũng ghi/đọc được nhanh (local region) trong khi dữ liệu vẫn được đồng bộ và giải quyết xung đột khi cùng field bị sửa ở 2 vùng gần như đồng thời.

**Yêu cầu cụ thể:**
- Định nghĩa rõ consistency model: active-active với eventual consistency, và nêu rõ time window tối đa dữ liệu có thể "lệch" giữa các vùng trước khi hội tụ.
- Có chiến lược giải quyết conflict rõ ràng cho field-level update (ví dụ last-write-wins theo hybrid logical clock, hoặc CRDT cho các field dạng counter/set).
- Xử lý network partition giữa 2 region kéo dài (region mất kết nối vài giờ): mỗi vùng vẫn phục vụ ghi/đọc local, và khi kết nối lại phải merge dữ liệu đã phân kỳ mà không mất update nào của cả 2 phía.
- Có audit log ghi lại các lần conflict thực sự xảy ra (cùng field bị sửa xung đột) và cách hệ thống đã giải quyết, để hỗ trợ debug/khiếu nại từ user.
- Đo lường độ trễ hội tụ (convergence time) trung bình và p99 giữa các vùng sau một lần ghi.

---

## Collaborative document editor đồng bộ chỉnh sửa xuyên vùng (kiểu Notion/Google Docs)

**Repository:** `multi-region-replication-collaborative-editor`

**Hệ thống:** Ứng dụng soạn tài liệu cộng tác thời gian thực, người dùng ở nhiều vùng địa lý cùng sửa một tài liệu.

**Vai trò của flow:** Đồng bộ thay đổi nội dung tài liệu giữa các vùng với độ trễ thấp nhất có thể, giải quyết xung đột chỉnh sửa đồng thời mà không làm mất nội dung của bất kỳ ai.

**Yêu cầu cụ thể:**
- Dùng cấu trúc dữ liệu hội tụ được (CRDT hoặc OT) để merge các thay đổi đồng thời từ nhiều người ở nhiều vùng mà không cần "khóa" tài liệu.
- Đảm bảo không có thao tác chỉnh sửa nào bị mất khi 2 người gõ vào cùng vị trí gần như đồng thời từ 2 region khác nhau — kết quả merge phải hợp lý (cả hai đoạn text đều xuất hiện theo thứ tự nhất quán ở mọi client).
- Xử lý được trường hợp một client mất kết nối và soạn offline một khoảng thời gian, khi online lại phải merge đúng các thay đổi offline vào bản mới nhất mà không revert thay đổi của người khác.
- Có cơ chế snapshot/checkpoint định kỳ để không phải replay toàn bộ lịch sử operation từ đầu mỗi khi tài liệu được mở, đặc biệt với tài liệu đã có lịch sử chỉnh sửa dài.
- Đo được độ trễ hiển thị thay đổi của người dùng ở vùng xa nhất (ví dụ người ở APAC thấy thay đổi của người ở US sau bao lâu) và tối ưu theo hướng chọn region gần nhất để relay.

---

## Đồng bộ tồn kho toàn cầu cho hệ thống e-commerce đa vùng

**Repository:** `multi-region-replication-ecommerce-inventory`

**Hệ thống:** Chuỗi bán lẻ online vận hành warehouse và bán hàng ở nhiều vùng, cần một view tồn kho toàn cục nhưng vẫn cho phép bán hàng nhanh ở local.

**Vai trò của flow:** Đồng bộ số lượng tồn kho giữa các vùng, xử lý xung đột khi tồn kho gần hết và nhiều vùng cùng bán sản phẩm cuối cùng gần như đồng thời.

**Yêu cầu cụ thể:**
- Không cho phép oversell toàn cục: dù mỗi vùng xử lý bán hàng local trước, tổng số bán ra không được vượt tồn kho thực tế — nêu cơ chế cụ thể (ví dụ reserve buffer theo vùng, sync định kỳ, hoặc quorum check cho SKU sắp hết hàng).
- Với SKU dư dả (stock lớn), ưu tiên tốc độ: cho phép eventual consistency, đồng bộ bất đồng bộ giữa vùng.
- Với SKU gần hết hàng (dưới ngưỡng an toàn), phải chuyển sang đồng bộ chặt hơn (đồng bộ/quorum) để giảm rủi ro oversell, đánh đổi lấy latency cao hơn.
- Khi phát hiện đã oversell do lệch đồng bộ (trường hợp xấu nhất), phải có luồng xử lý hậu kỳ tự động (báo khách trễ hàng/hủy đơn/đề xuất sản phẩm thay thế) chứ không im lặng.
- Có dashboard theo dõi độ lệch tồn kho giữa các vùng theo thời gian thực và cảnh báo khi độ lệch vượt ngưỡng.

---

## Đồng bộ số liệu like/counter trên mạng xã hội toàn cầu

**Repository:** `multi-region-replication-social-like-counter`

**Hệ thống:** Nền tảng mạng xã hội có bài viết được like/react từ người dùng ở khắp thế giới, số liệu được ghi phân tán ở nhiều vùng.

**Vai trò của flow:** Đồng bộ và cộng dồn số lượng like giữa các vùng mà không cần lock toàn cục, tối ưu cho throughput cao và độ trễ thấp, đánh đổi độ chính xác tuyệt đối tức thời.

**Yêu cầu cụ thể:**
- Dùng cấu trúc dữ liệu dạng CRDT counter (ví dụ PN-Counter) để mỗi vùng tăng/giảm độc lập và tổng hợp lại một cách hội tụ, không cần đồng bộ đồng thời.
- Số liệu hiển thị cho user có thể là "gần đúng" (eventually consistent) trong vài giây, nhưng phải nêu rõ ngưỡng thời gian tối đa để hội tụ về đúng số thực.
- Chống double-count khi cùng một user like rồi unlike rất nhanh từ 2 thiết bị/2 vùng khác nhau — đảm bảo trạng thái cuối cùng phản ánh đúng hành động cuối cùng của user đó.
- Xử lý mất kết nối giữa vùng kéo dài: mỗi vùng vẫn tiếp tục nhận like local, và khi nối lại phải merge không làm tăng ảo số lượng like (không đếm lại từ đầu).
- Đo lường: throughput like/giây tối đa hệ thống chịu được theo từng vùng, và độ trễ hội tụ số liệu giữa vùng xa nhau nhất.

---

## Đảm bảo thứ tự tin nhắn trong chat app đa vùng

**Repository:** `multi-region-replication-chat-message-ordering`

**Hệ thống:** App chat/messenger có user ở nhiều vùng, message được relay qua các data center gần user nhất để giảm latency gửi/nhận.

**Vai trò của flow:** Đảm bảo thứ tự hiển thị tin nhắn hợp lý (nhất quán theo causal order) cho tất cả người trong cùng cuộc chat, dù message được ghi vào các vùng khác nhau.

**Yêu cầu cụ thể:**
- Dùng cơ chế đánh thứ tự logic (ví dụ vector clock hoặc hybrid logical clock) để xác định thứ tự causal giữa các message, không chỉ dựa vào timestamp đồng hồ vật lý (có thể lệch giữa server các vùng).
- Đảm bảo message trả lời (reply) luôn hiển thị sau message được trả lời, dù 2 message được ghi ở 2 vùng khác nhau và tới đích theo thứ tự vật lý khác.
- Xử lý mất kết nối tạm thời giữa vùng: message gửi trong lúc mất kết nối phải được buffer và đồng bộ đúng thứ tự khi kết nối phục hồi, không hiển thị đảo lộn cho người nhận.
- Cho phép trường hợp 2 người gõ và gửi gần như đồng thời (không có quan hệ nhân-quả) hiển thị theo thứ tự nhất quán nhưng không cần đúng tuyệt đối theo thời gian thực — miễn là mọi client thấy CÙNG một thứ tự.
- Đo lường độ trễ gửi/nhận tin nhắn xuyên vùng (round-trip) và tỉ lệ tin nhắn phải "reorder" khi hiển thị do tới muộn.
