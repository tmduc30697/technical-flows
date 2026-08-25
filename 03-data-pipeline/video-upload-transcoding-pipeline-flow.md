# Video upload & transcoding pipeline flow — Đề bài thực hành

Các đề bài dưới đây trải qua nhiều bối cảnh web khác nhau — nền tảng chia sẻ video, e-learning, short-video mobile, marketplace, SaaS B2B webinar, và hậu xử lý livestream — nhằm luyện đủ các góc của flow upload & transcode video (chunking, hàng đợi xử lý, đa độ phân giải, xử lý lỗi, chi phí vận hành).

---

## Nền tảng chia sẻ video kiểu YouTube

**Repository:** `video-transcoding-youtube-like-sharing`

**Hệ thống:** Một nền tảng cho phép user upload video và người khác xem lại (video-on-demand), quy mô hàng triệu video.

**Vai trò của flow:** Nhận file video gốc từ user, transcode ra nhiều độ phân giải/bitrate và đóng gói HLS/DASH để phát trên mọi thiết bị.

**Yêu cầu cụ thể:**
- Upload phải hỗ trợ file lớn (nhiều GB) bằng chunked/resumable upload, tiếp tục được nếu mất mạng giữa chừng.
- Sau khi upload xong, đưa vào hàng đợi transcode ra tối thiểu 3 độ phân giải (ví dụ 1080p/720p/480p) chạy song song, không block nhau nếu một rendition lỗi.
- Sinh thumbnail tự động (nhiều frame để user chọn) và file manifest HLS/DASH tương ứng.
- Nếu job transcode fail giữa chừng (worker crash, hết dung lượng tạm), phải retry lại đúng phần việc còn thiếu, không transcode lại từ đầu toàn bộ video.
- Video chỉ hiển thị "public" sau khi tối thiểu 1 rendition sẵn sàng; các rendition còn lại có thể hoàn thành dần (progressive availability).
- Có cơ chế dọn file gốc/file tạm sau X ngày để tối ưu chi phí storage, nhưng vẫn giữ được khả năng re-transcode nếu cần đổi codec sau này.

---

## Nền tảng e-learning với video bài giảng

**Repository:** `video-transcoding-elearning-lecture`

**Hệ thống:** Một SaaS cho giảng viên upload video bài giảng, học viên xem theo khóa học, có yêu cầu phụ đề.

**Vai trò của flow:** Xử lý video bài giảng thành định dạng streaming tối ưu, kèm sinh phụ đề tự động và gắn watermark chống lộ nội dung khóa học.

**Yêu cầu cụ thể:**
- Pipeline phải chèn bước tạo phụ đề tự động (speech-to-text) chạy song song với transcode hình ảnh, rồi merge kết quả trước khi publish.
- Video chỉ được publish cho học viên xem khi giảng viên đã duyệt (preview) bản đã transcode — cho phép giảng viên yêu cầu transcode lại nếu chất lượng không đạt.
- Gắn watermark động (email/tên học viên) vào từng bản stream để chống chia sẻ lại — không được làm tăng đáng kể thời gian xử lý tổng thể.
- Giới hạn tổng số phút video xử lý mỗi tháng theo gói cước của giảng viên; job phải bị chặn/queue lại rõ ràng nếu vượt quota, không âm thầm fail.
- Xử lý được trường hợp giảng viên upload nhiều video cùng lúc cho một khóa học — đảm bảo thứ tự publish đúng theo thứ tự bài học dù xử lý xong không đồng thời.

---

## App video ngắn dạng TikTok

**Repository:** `video-transcoding-short-video-tiktok-like`

**Hệ thống:** Một mạng xã hội video ngắn, upload chủ yếu từ mobile, yêu cầu video xuất hiện gần như ngay lập tức sau khi đăng.

**Vai trò của flow:** Transcode nhanh video ngắn (dưới 60s) từ nhiều codec/tỉ lệ khung hình mobile khác nhau thành định dạng chuẩn hóa, tối ưu độ trễ end-to-end.

**Yêu cầu cụ thể:**
- Thời gian từ lúc upload xong đến lúc video "có thể xem được" (ít nhất 1 rendition) phải đạt SLA vài giây đến vài chục giây, khác hẳn pipeline video dài — cần ưu tiên hàng đợi riêng cho video ngắn.
- Chuẩn hóa được input từ nhiều tỉ lệ khung hình (9:16, 1:1, 16:9) và framerate khác nhau về cùng chuẩn output, không làm méo hoặc crop sai nội dung.
- Video phải đi qua bước kiểm tra sơ bộ (duration, kích thước, định dạng hợp lệ) trước khi vào hàng đợi transcode, từ chối sớm các file không hợp lệ để tránh tốn tài nguyên worker.
- Khi lượng upload tăng đột biến (viral moment, giờ cao điểm), hệ thống phải tự scale worker transcode theo hàng đợi, có cơ chế backpressure để không làm sập toàn bộ hệ thống.
- Video bị người dùng xóa trong lúc đang transcode phải hủy job giữa chừng và dọn sạch file trung gian, không tiếp tục xử lý lãng phí.

---

## Marketplace bán hàng có video demo sản phẩm

**Repository:** `video-transcoding-marketplace-product-demo`

**Hệ thống:** Một nền tảng e-commerce cho phép seller upload video demo/unbox sản phẩm gắn vào trang listing.

**Vai trò của flow:** Xử lý video seller upload thành định dạng chuẩn để hiển thị trên trang sản phẩm, tích hợp với bước kiểm duyệt nội dung trước khi public.

**Yêu cầu cụ thể:**
- Video sau khi transcode xong phải được giữ ở trạng thái "chờ duyệt" và chỉ gắn vào listing công khai sau khi qua bước kiểm duyệt nội dung (không quảng cáo sai, không nội dung cấm).
- Hỗ trợ seller upload hàng loạt (bulk) nhiều video cho nhiều sản phẩm cùng lúc; hệ thống phải theo dõi và báo cáo trạng thái xử lý riêng cho từng video, không gộp chung một trạng thái.
- Giới hạn độ dài và kích thước file theo policy (ví dụ tối đa 2 phút, 200MB) — từ chối ngay ở bước upload, không đưa vào hàng đợi transcode rồi mới từ chối.
- Khi seller thay video mới cho listing đã có video cũ, phải đảm bảo không có khoảng thời gian listing hiển thị video lỗi/trống trong lúc video mới đang xử lý (giữ video cũ hiển thị đến khi video mới sẵn sàng).
- Theo dõi chi phí transcode theo từng seller/gói để tính vào chi phí vận hành, cảnh báo nếu một seller tiêu tốn tài nguyên bất thường (có thể là spam/abuse).

---

## SaaS B2B lưu trữ video hội thảo/webinar cho doanh nghiệp

**Repository:** `video-transcoding-b2b-saas-webinar-storage`

**Hệ thống:** Một SaaS cho nhiều công ty khách hàng (multi-tenant) upload và lưu trữ video hội thảo nội bộ, phân quyền xem theo tổ chức.

**Vai trò của flow:** Transcode video hội thảo thành các rendition phù hợp với băng thông văn phòng, đảm bảo cách ly hoàn toàn dữ liệu giữa các tenant.

**Yêu cầu cụ thể:**
- Toàn bộ file gốc, file trung gian và output transcode phải được cách ly theo tenant (storage path/namespace riêng), không rò rỉ chéo giữa các công ty dù dùng chung hạ tầng xử lý.
- Mỗi tenant có hạn mức storage và số phút xử lý theo hợp đồng; hệ thống phải tính toán và cảnh báo trước khi tenant vượt hạn mức, có cơ chế từ chối job mới một cách rõ ràng khi vượt.
- Cho phép admin của từng tenant cấu hình độ phân giải tối đa cần transcode (một số công ty không cần 4K) để tối ưu chi phí — pipeline phải đọc config này trước khi khởi tạo job.
- Video nội bộ nhạy cảm phải hỗ trợ mã hóa tại rest và cấp link xem có thời hạn (signed URL hết hạn sau X giờ), không cho phép link bị chia sẻ vô thời hạn.
- Cung cấp báo cáo chi phí xử lý (phút video đã transcode, dung lượng lưu trữ) theo từng tenant để phục vụ billing nội bộ hoặc chargeback.

---

## Hậu xử lý bản ghi sau khi kết thúc livestream

**Repository:** `video-transcoding-livestream-vod-postprocessing`

**Hệ thống:** Một nền tảng livestream cho phép người xem xem lại (VOD) sau khi buổi live kết thúc.

**Vai trò của flow:** Nhận file ghi hình thô (đã ghép từ các segment trong lúc live) và chạy transcode thành VOD chất lượng cao, khác với luồng xử lý real-time lúc đang live.

**Yêu cầu cụ thể:**
- Khi stream kết thúc, hệ thống phải ghép các segment ghi hình (thường ở dạng .ts nhỏ) thành file liên tục chính xác theo thứ tự thời gian, xử lý được trường hợp segment bị thiếu/lỗi do gián đoạn mạng lúc live.
- Chạy transcode lại toàn bộ VOD ở chất lượng cao hơn bản live gốc (vì không còn ràng buộc độ trễ thấp), tận dụng được thời gian off-peak để giảm chi phí worker.
- VOD phải giữ được đúng các mốc thời gian quan trọng trong lúc live (ví dụ thời điểm donate, highlight được đánh dấu) sau khi ghép và transcode.
- Nếu buổi live kéo dài nhiều giờ, pipeline phải xử lý theo từng đoạn (chunked processing) thay vì chờ xử lý toàn bộ một lần, để VOD có thể sẵn sàng dần (phần đầu xem được trước khi phần cuối xử lý xong).
- Có cơ chế dọn dữ liệu ghi hình thô sau khi VOD đã tạo thành công và được xác nhận phát được, tránh giữ trùng lặp dữ liệu gây tốn chi phí lưu trữ lâu dài.
