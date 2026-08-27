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

---

## E-learning với phụ đề và chương mục đồng bộ

**Repository:** `video-transcoding-elearning-subtitle-chapter-sync`

**Hệ thống:** Một nền tảng e-learning, video bài giảng cần đi kèm phụ đề và chương mục (chapter markers) để học viên tra cứu nhanh nội dung.

**Vai trò của flow:** Transcode video bài giảng ra nhiều độ phân giải đồng thời xử lý và gắn phụ đề/mốc chương mục, đảm bảo các mốc thời gian này luôn khớp chính xác với video sau khi qua các bước xử lý.

**Yêu cầu cụ thể:**
- Khi video được transcode qua nhiều bước (cắt intro thừa, chuẩn hóa framerate, đổi tốc độ phát nếu giảng viên upload bản đã chỉnh sửa), các mốc timestamp của phụ đề và chapter marker phải được tính lại đồng bộ theo từng bước biến đổi, tránh tình trạng phụ đề bị lệch dần so với hình sau khi qua pipeline xử lý.
- Nếu giảng viên upload file phụ đề riêng sau khi video đã transcode xong và đã publish cho học viên xem, việc gắn thêm phụ đề mới không được yêu cầu transcode lại toàn bộ video, mà chỉ cần xử lý và đồng bộ lại phần phụ đề độc lập với các rendition video đã có sẵn.
- Chapter marker do giảng viên đánh dấu thủ công tại thời điểm xem bản gốc trước khi transcode có thể lệch nếu quá trình transcode làm thay đổi độ dài tổng thể video (ví dụ loại bỏ đoạn câm/đen ở đầu) — cần cơ chế ánh xạ lại chính xác mốc thời gian gốc sang mốc thời gian của video đã transcode, không đơn giản giữ nguyên số giây gốc.
- Với khóa học có video dài, việc xử lý phụ đề/chapter không được làm chậm thời gian video sẵn sàng để học viên xem — cần tách pipeline xử lý phụ đề/chapter chạy song song độc lập với pipeline transcode video chính, cho phép video hiển thị trước và phụ đề/chapter cập nhật bổ sung sau nếu xử lý xong muộn hơn.
- Khi giảng viên chỉnh sửa lại nội dung phụ đề sau khi khóa học đã có học viên đang học, bản phụ đề cũ đang được học viên xem dở không được thay đổi đột ngột giữa chừng gây gián đoạn trải nghiệm, cần có cơ chế áp dụng bản cập nhật hợp lý (ví dụ áp dụng từ lần xem tiếp theo).

---

## Marketplace với video sản phẩm tự upload từ seller

**Repository:** `video-transcoding-marketplace-seller-listing-video`

**Hệ thống:** Một marketplace cho phép seller tự upload video giới thiệu sản phẩm khi đăng tin bán, chất lượng/định dạng đầu vào rất khác nhau tùy thiết bị quay (điện thoại, máy quay chuyên nghiệp).

**Vai trò của flow:** Nhận video từ seller, chuẩn hóa về định dạng/chất lượng thống nhất để hiển thị trên trang sản phẩm, không làm chậm quá trình đăng tin bán hàng của seller.

**Yêu cầu cụ thể:**
- Video đầu vào có thể ở đủ loại codec, độ phân giải, tỉ lệ khung hình, và cả hướng quay tùy thiết bị seller dùng — pipeline chuẩn hóa phải nhận diện đúng và xử lý được sự đa dạng này thành output nhất quán trên trang sản phẩm mà không yêu cầu seller phải tự chỉnh sửa/convert trước khi upload.
- Việc seller đăng tin bán không nên bị chặn chờ video xử lý xong hoàn toàn — tin sản phẩm phải hiển thị được ngay với ảnh/thông tin khác trong khi video còn đang transcode ở nền, và tự động cập nhật hiển thị video khi xử lý xong mà không cần seller thao tác lại.
- Một số video từ seller cá nhân dùng điện thoại quay ở chất lượng thấp/rung/tối — hệ thống cần có ngưỡng kiểm tra chất lượng đầu vào tối thiểu để từ chối sớm hoặc cảnh báo seller thay vì lãng phí tài nguyên transcode ra một video chất lượng kém không cải thiện được gì so với gốc.
- Khi seller upload video mới để thay thế video cũ của một tin đang bán chạy, việc thay thế phải diễn ra mà không tạo khoảng trống hiển thị (tin tạm thời không có video) hoặc hiển thị nhầm video cũ/video mới lẫn lộn cho các khách đang xem tin cùng lúc.
- Với lượng seller lớn upload đồng thời ở giờ cao điểm đăng tin, hàng đợi transcode phải đảm bảo công bằng giữa các seller (tránh một seller upload nhiều video liên tục chiếm hết tài nguyên khiến seller khác phải chờ lâu bất thường), đồng thời kiểm soát được chi phí xử lý khi khối lượng video từ seller cá nhân thường nhỏ lẻ nhưng số lượng rất lớn.

---

## SaaS B2B webinar với yêu cầu gửi bản ghi ngay sau buổi họp

**Repository:** `video-transcoding-saas-b2b-webinar-fast-delivery`

**Hệ thống:** Một SaaS B2B tổ chức webinar/họp video với khách hàng, cần gửi bản ghi lại cho khách hàng ngay sau khi buổi họp kết thúc, một số trường hợp cần tách riêng audio/video theo từng người nói.

**Vai trò của flow:** Nhận file ghi hình buổi họp/webinar ngay sau khi kết thúc, transcode và xử lý ưu tiên tốc độ để có thể gửi cho khách hàng trong thời gian ngắn nhất, kèm khả năng tách nguồn theo người nói nếu được yêu cầu.

**Yêu cầu cụ thể:**
- Thời gian từ lúc buổi họp kết thúc đến lúc bản ghi sẵn sàng gửi cho khách hàng phải rất ngắn để giữ được ngữ cảnh còn "nóng" của cuộc họp — pipeline này cần ưu tiên tài nguyên xử lý cao hơn hẳn so với video thông thường không có ràng buộc thời gian, kể cả phải đánh đổi chi phí compute cao hơn cho một phiên xử lý ưu tiên.
- Nếu buổi họp có nhiều người nói được ghi trên các track/nguồn audio riêng biệt, việc ghép và đồng bộ các track này với video chung phải chính xác đến từng khung hình để tránh audio người này chồng lấn/lệch với hình ảnh người khác đang nói.
- Trường hợp buổi họp bị dừng đột ngột giữa chừng hoặc file ghi hình bị lỗi một phần, hệ thống phải xử lý được phần dữ liệu còn dùng được để gửi bản ghi một phần cho khách hàng kèm thông báo rõ ràng, thay vì để khách hàng chờ vô thời hạn một bản ghi sẽ không bao giờ hoàn chỉnh.
- Nội dung webinar/họp thường chứa thông tin nhạy cảm giữa hai doanh nghiệp — bản ghi trong lúc đang xử lý và file trung gian phải được cô lập đúng theo từng khách hàng/tổ chức, không để rò rỉ chéo giữa hàng đợi xử lý của các khách hàng khác nhau kể cả khi chạy trên cùng hạ tầng xử lý dùng chung.
- Khi có nhu cầu tách riêng audio/video theo từng người nói để phục vụ nhu cầu khác nhau của khách hàng, việc tách này không được yêu cầu xử lý lại từ đầu toàn bộ file gốc mỗi lần có yêu cầu mới, mà nên tận dụng lại kết quả trung gian đã tách nguồn từ lần xử lý ban đầu.
