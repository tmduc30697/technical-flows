# Message broker tự xây đồng thuận thứ tự message

**Hệ thống:** Một message broker tự xây (không dùng lại broker có sẵn) gồm nhiều broker node, phục vụ publish/subscribe cho nhiều consumer, cần đảm bảo consumer nhận message theo đúng thứ tự đã publish dù cụm broker có node chết hoặc leader broker bị thay đổi.

**Vai trò của flow:** Consensus giữa các broker node quyết định thứ tự chính thức (canonical order) của message trong một topic/partition, đảm bảo khi leader broker hiện tại chết và một broker khác lên thay, thứ tự message consumer nhận vẫn nhất quán và không bị mất/lặp message đã được xác nhận.

**Yêu cầu cụ thể:**
- Mọi message publish vào một topic/partition phải được gán thứ tự (offset) chỉ sau khi broker leader hiện tại đã replicate thành công tới majority broker node khác trong nhóm phụ trách partition đó — message chỉ được coi là "đã publish thành công" (ack cho producer) sau bước này, không phải ngay khi leader nhận được.
- Khi broker leader phụ trách một partition chết đột ngột, broker mới lên thay phải xác định chính xác offset cuối cùng đã thực sự commit (replicate đủ majority) trước khi cho phép publish tiếp — nếu chọn nhầm một offset dựa trên log chưa commit đầy đủ của leader cũ, có thể làm mất message đã ack cho producer hoặc gây phân nhánh thứ tự giữa các replica.
- Trong lúc leader mới đang được bầu (giai đoạn gián đoạn ngắn), phải quyết định rõ hành vi cho producer đang cố publish (từ chối rõ ràng để producer tự retry, hay buffer tạm ở phía client) và cho consumer đang đọc (tạm dừng nhận message mới ở đúng offset cuối cùng đã đọc, không được đọc nhảy cóc hoặc đọc trùng khi kết nối lại broker mới).
- Consumer group có thể đang đọc dở một partition đúng lúc leader chuyển giao — cần đảm bảo offset commit của consumer (điểm đã đọc tới đâu) được lưu độc lập với trạng thái broker leader, để khi consumer reconnect vào broker mới, nó tiếp tục đọc đúng từ vị trí đã dừng, không bị mất message (đọc thiếu) hoặc nhận lại message đã xử lý (đọc trùng) một cách không kiểm soát được.
- Đo lường: thời gian gián đoạn publish/consume trong mỗi lần chuyển leader (failover time), tần suất phải chuyển leader trong thực tế vận hành, và có test định kỳ mô phỏng chết leader giữa lúc đang publish tải cao để xác nhận không có message nào bị mất hoặc đảo thứ tự ngoài dự kiến.
