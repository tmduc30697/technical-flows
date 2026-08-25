# Content moderation flow (AI + human review pipeline) — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — mạng xã hội, marketplace, app hẹn hò, livestream chat, nền tảng video, và marketplace freelancer — nhằm luyện đủ các góc của flow kiểm duyệt nội dung kết hợp AI và người kiểm duyệt (hàng đợi ưu tiên, độ trễ, false positive/negative, kháng cáo, chi phí vận hành review).

---

## Mạng xã hội kiểm duyệt bài viết/hình ảnh do người dùng đăng

**Repository:** `content-moderation-social-post-image`

**Hệ thống:** Một mạng xã hội cho phép user đăng bài viết kèm hình ảnh, có lượng nội dung đăng mới rất lớn mỗi giờ.

**Vai trò của flow:** Lọc tự động nội dung vi phạm bằng AI ngay khi đăng, chỉ đưa các trường hợp AI không chắc chắn vào hàng đợi cho người kiểm duyệt xử lý.

**Yêu cầu cụ thể:**
- AI classifier phải trả về mức độ tin cậy (confidence score); nội dung có điểm vi phạm rất cao bị chặn/ẩn ngay, điểm rất thấp được publish ngay, chỉ vùng "không chắc chắn" ở giữa mới vào hàng đợi người duyệt.
- Hàng đợi cho người kiểm duyệt phải ưu tiên theo mức độ nghiêm trọng (nội dung nghi ngờ bạo lực/tự hại phải được ưu tiên xử lý trước nội dung spam thông thường).
- Trong thời gian chờ người duyệt (đối với nội dung ở vùng không chắc chắn), phải quyết định rõ chính sách hiển thị tạm thời (ẩn trước-duyệt sau, hoặc hiển thị hạn chế) và áp dụng nhất quán.
- Người dùng bị gỡ bài phải có cơ chế kháng cáo, kháng cáo được đưa vào một hàng đợi review riêng (khác luồng kiểm duyệt lần đầu) và phải có SLA thời gian phản hồi.
- Ghi lại đầy đủ quyết định của AI và của người duyệt (lý do, model version, người duyệt nào) để phục vụ audit và để tiếp tục huấn luyện lại model theo thời gian.
- Hệ thống phải chịu được lượng nội dung tăng đột biến (breaking event, viral topic) mà không làm tăng thời gian xử lý hàng đợi vượt SLA đã cam kết.

---

## Marketplace kiểm duyệt tin đăng bán sản phẩm của seller

**Repository:** `content-moderation-marketplace-listing`

**Hệ thống:** Một sàn marketplace cho phép seller tự đăng sản phẩm lên bán, cần kiểm duyệt trước khi hiển thị công khai để tránh hàng giả, hàng cấm, quảng cáo sai sự thật.

**Vai trò của flow:** Kiểm tra tin đăng mới (text, ảnh, giá) bằng AI để phát hiện vi phạm phổ biến, chuyển các trường hợp nghi ngờ cho nhân viên kiểm duyệt xác nhận trước khi tin đăng được public.

**Yêu cầu cụ thể:**
- Tin đăng mới phải ở trạng thái "chờ duyệt" (không hiển thị cho khách mua) cho đến khi qua được bước kiểm tra tự động hoặc người duyệt xác nhận — không public trước rồi gỡ sau.
- AI phải phát hiện được các dấu hiệu phổ biến: từ khóa hàng cấm, ảnh sản phẩm trùng với ảnh đã bị gỡ trước đó (nghi hàng giả lặp lại), giá bất thường so với sản phẩm cùng loại.
- Seller có lịch sử vi phạm nhiều lần phải được đưa vào luồng kiểm duyệt chặt hơn (review 100% bởi người, không qua auto-approve) cho các tin đăng tiếp theo, cho đến khi được gỡ trạng thái nghi ngờ.
- Thời gian chờ duyệt trung bình phải được đo và tối ưu vì ảnh hưởng trực tiếp đến trải nghiệm seller; cần có cơ chế auto-approve cho seller uy tín cao với loại sản phẩm rủi ro thấp để giảm tải cho người duyệt.
- Khi một tin đăng đã public bị phát hiện vi phạm sau đó (báo cáo từ người dùng khác), phải có luồng xử lý riêng (post-publish moderation) khác với luồng duyệt trước khi đăng, và phải ẩn ngay trong lúc chờ xác minh.

---

## App hẹn hò kiểm duyệt ảnh và hồ sơ người dùng

**Repository:** `content-moderation-dating-app-profile`

**Hệ thống:** Một app hẹn hò yêu cầu user upload ảnh đại diện và viết bio, cần đảm bảo không có nội dung khiêu dâm, tài khoản giả, hoặc lừa đảo.

**Vai trò của flow:** Kiểm duyệt ảnh và bio ngay khi user tạo/cập nhật hồ sơ, kết hợp phát hiện tự động và xác minh của người kiểm duyệt trước khi hồ sơ được hiển thị cho người dùng khác.

**Yêu cầu cụ thể:**
- Ảnh đại diện phải qua bước phát hiện nội dung khiêu dâm/nhạy cảm tự động ngay khi upload, chặn ngay các trường hợp rõ ràng vi phạm mà không cần chờ người duyệt.
- Phải có cơ chế phát hiện ảnh giả mạo/đánh cắp (ví dụ ảnh lấy từ nguồn công khai khác, ảnh của người nổi tiếng) để giảm tài khoản lừa đảo (catfishing) — các trường hợp nghi ngờ chuyển cho người duyệt xác minh thủ công.
- Bio text phải được kiểm tra để phát hiện thông tin liên hệ trái phép (số điện thoại, link ngoài) thường dùng để lách qua kênh chat trong app nhằm lừa đảo, và chặn/cảnh báo phù hợp.
- Trong thời gian hồ sơ đang chờ duyệt lần đầu, user vẫn phải trải nghiệm được các phần khác của app (không bị block hoàn toàn), nhưng hồ sơ chưa hiển thị cho người khác ghép đôi.
- Khi có báo cáo từ người dùng khác về một hồ sơ đã public (nghi ngờ giả/lừa đảo), luồng xử lý phải ưu tiên hồ sơ đó vào hàng đợi review khẩn, có thể tạm ẩn hồ sơ trong lúc chờ xác minh nếu số lượng báo cáo vượt ngưỡng.

---

## Kiểm duyệt chat trong lúc livestream (real-time)

**Repository:** `content-moderation-livestream-chat-realtime`

**Hệ thống:** Một nền tảng livestream có khung chat công khai cho hàng nghìn người xem cùng gõ chat trong lúc streamer đang phát trực tiếp.

**Vai trò của flow:** Lọc tin nhắn spam/độc hại theo thời gian thực trước khi hiển thị lên màn hình chat chung, với độ trễ tối thiểu để không làm chậm trải nghiệm chat trực tiếp.

**Yêu cầu cụ thể:**
- Việc kiểm duyệt tin nhắn phải diễn ra trong vòng chưa đến một giây để không cảm nhận được độ trễ rõ rệt trong luồng chat trực tiếp — không thể áp dụng mô hình "chờ người duyệt" như nội dung tĩnh.
- Phải xử lý được tấn công spam hàng loạt (bot gửi hàng trăm tin nhắn giống nhau trong vài giây) bằng rate-limit và phát hiện pattern lặp, không để một tài khoản làm ngập toàn bộ khung chat.
- Từ điển/luật lọc từ ngữ độc hại phải cấu hình được riêng theo từng streamer/kênh (một số streamer cho phép ngôn ngữ thoải mái hơn), không dùng một bộ luật cứng chung cho toàn nền tảng.
- Tin nhắn bị AI nghi ngờ nhưng không chắc chắn (borderline) không được giữ lại để "người duyệt xem sau rồi hiện" vì chat đã trôi qua — hệ thống phải quyết định ẩn/hiện ngay tại thời điểm gửi, và ghi log để hậu kiểm, xử phạt tài khoản nếu cần.
- Moderator (có thể là người do streamer chỉ định) phải có công cụ xóa tin nhắn/cấm chat theo thời gian thực, và hành động đó phải phản ánh ngay trên toàn bộ client đang xem, không có độ trễ đáng kể.

---

## Nền tảng video kiểm duyệt nội dung sau khi upload, trước khi public toàn nền tảng

**Repository:** `content-moderation-video-post-upload`

**Hệ thống:** Một nền tảng chia sẻ video cho phép ai cũng upload, cần kiểm duyệt nội dung (bạo lực, bản quyền, thông tin sai lệch) trước khi video được đề xuất rộng rãi.

**Vai trò của flow:** Sau khi video được transcode xong, chạy qua pipeline kiểm duyệt đa lớp (phân tích hình ảnh theo frame, âm thanh, phụ đề/transcript) rồi mới quyết định mức độ phân phối của video.

**Yêu cầu cụ thể:**
- Kiểm duyệt phải phân tích được nội dung theo nhiều tầng: hình ảnh (frame sampling), âm thanh, và transcript từ giọng nói — kết hợp kết quả của cả ba tầng để ra quyết định cuối, không chỉ dựa vào một tầng duy nhất.
- Video có thể được publish ngay ở trạng thái "hiển thị hạn chế" (chỉ người theo dõi trực tiếp xem được, không được đề xuất) trong lúc kiểm duyệt đầy đủ đang chạy, để không làm chậm trải nghiệm creator, và mở rộng phân phối sau khi qua kiểm duyệt.
- Phải phát hiện được nội dung vi phạm bản quyền qua so khớp fingerprint âm thanh/hình ảnh với cơ sở dữ liệu nội dung đã đăng ký bản quyền, và có luồng xử lý riêng (khác luồng nội dung cấm) cho trường hợp này.
- Người kiểm duyệt phải xem được các đoạn/frame được AI đánh dấu nghi ngờ thay vì phải xem lại toàn bộ video từ đầu, để tối ưu thời gian xử lý cho video dài.
- Khi creator kháng cáo quyết định kiểm duyệt, phải có luồng xử lý kháng cáo riêng với người duyệt khác (không phải người đã ra quyết định ban đầu) để tránh thiên vị, và phải công khai lý do quyết định cho creator.

---

## Marketplace freelancer kiểm duyệt tin đăng công việc/gig và portfolio

**Repository:** `content-moderation-freelance-marketplace-gig`

**Hệ thống:** Một nền tảng kết nối freelancer và khách hàng thuê dịch vụ, cho phép freelancer đăng gig kèm portfolio và khách hàng đăng job.

**Vai trò của flow:** Kiểm duyệt cả hai chiều — gig/portfolio của freelancer và job đăng của khách hàng — để phát hiện lừa đảo, nội dung sao chép, hoặc dịch vụ vi phạm chính sách.

**Yêu cầu cụ thể:**
- Portfolio (ảnh/dự án mẫu) của freelancer phải được kiểm tra trùng lặp với portfolio của freelancer khác đã đăng trước đó (nghi ngờ đạo/sao chép sản phẩm người khác), đưa vào hàng đợi review nếu phát hiện độ tương đồng cao.
- Job đăng từ khách hàng phải được kiểm tra dấu hiệu lừa đảo phổ biến (yêu cầu freelancer trả phí trước, yêu cầu thông tin ngân hàng bất thường) và chặn/gắn cờ trước khi job hiển thị công khai cho freelancer ứng tuyển.
- Gig có mô tả dịch vụ nằm trong danh mục nhạy cảm (ví dụ liên quan tài chính, y tế, pháp lý) phải được định tuyến vào hàng đợi review chuyên biệt có tiêu chí duyệt khác với gig thông thường (thiết kế, viết content).
- Tin nhắn trao đổi giữa freelancer và khách hàng trước khi chốt hợp đồng phải được quét để phát hiện dấu hiệu cố ý giao dịch ngoài nền tảng (né phí hoa hồng), có ngưỡng cảnh báo cho đội vận hành mà không đọc/lưu trữ nội dung riêng tư quá mức cần thiết.
- Khi một freelancer hoặc khách hàng bị report nhiều lần từ các đối tác giao dịch khác nhau, hệ thống phải tự động nâng mức độ kiểm duyệt cho toàn bộ hoạt động tiếp theo của tài khoản đó cho đến khi được xác minh lại.
