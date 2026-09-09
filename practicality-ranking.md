# Xếp hạng mức độ thực tiễn của các đề bài

Toàn bộ 156 đề trong `exercises/` được đọc README và chấm điểm **mức độ thực tiễn** (1-10): xác suất một developer bình thường (backend/fullstack, đi làm ở nhiều loại công ty — startup, công ty vừa, tech lớn — không giới hạn 1 ngành cấp phép cụ thể) thực sự gặp đúng bài toán này trong sự nghiệp.

**Thang điểm:**
- **9-10** — Cực kỳ phổ biến, hầu hết dev backend nào cũng đụng tới, không cần công ty đặc thù.
- **7-8** — Phổ biến ở công ty có sản phẩm quy mô kha khá, cần đúng loại hình sản phẩm (ecommerce, SaaS B2B, mobile...).
- **5-6** — Trung bình, chỉ gặp nếu đúng ngành/đúng quy mô cụ thể.
- **3-4** — Hiếm, chỉ gặp ở công ty quy mô lớn/ngành đặc thù hoặc vai trò platform/infra chuyên biệt.
- **1-2** — Cực hiếm, chỉ gặp ở ngành bị hạn chế giấy phép (ngân hàng lõi, sàn crypto, y tế cấp phép, cờ bạc) hoặc đội tự xây hạ tầng lõi (tự viết DB engine, tự cài Raft...) — tuyệt đại đa số dev cả sự nghiệp không đụng tới.

Sắp xếp giảm dần theo điểm; trong cùng 1 mức điểm, xếp theo alphabet. Danh sách được chấm bởi 6 lượt đọc README song song, có thể có sai lệch nhỏ ±1 điểm giữa các đề do đánh giá chủ quan — coi đây là kim chỉ nam tương đối, không phải con số tuyệt đối.

**Tổng quan số lượng theo tier:** 9đ: 7 đề · 8đ: 12 đề · 7đ: 17 đề · 6đ: 29 đề · 5đ: 12 đề · 4đ: 29 đề · 3đ: 30 đề · 2đ: 14 đề · 1đ: 6 đề

---

## Điểm 9 — Cực kỳ phổ biến

| Đề bài | Lý do |
|---|---|
| `account-recovery-marketplace-email-reset` | Luồng quên mật khẩu qua email tiêu chuẩn (chống enumeration, token, rate-limit) gần như mọi backend dev đều làm qua. |
| `cache-invalidation-ecommerce-product-page` | Cache invalidation cho trang sản phẩm ecommerce là ví dụ kinh điển gần như mọi dev backend traffic cao đều từng đụng. |
| `flash-sale-ecommerce-limited-stock` | Bài toán kinh điển "atomic decrement chống oversell" mà hầu như dev ecommerce nào cũng từng đụng tới. |
| `graceful-shutdown-background-worker-queue` | Graceful shutdown cho worker consume queue (không mất/không lặp message) gần như mọi hệ thống có background job đều cần. |
| `locking-cms-concurrent-editing` | Optimistic locking bằng version là pattern nền tảng gặp ở hầu hết mọi ứng dụng CRUD nhiều người dùng cùng sửa. |
| `oauth-google-login-internal-dashboard` | "Sign in with Google" bằng authorization code flow cực kỳ phổ biến ở hầu như mọi dự án nội bộ/SaaS. |
| `rate-limiting-login-brute-force` | Chống brute-force login là yêu cầu bảo mật cơ bản mà gần như mọi web app có đăng nhập đều cần. |

## Điểm 8

| Đề bài | Lý do |
|---|---|
| `circuit-breaker-checkout-payment-gateway` | Circuit breaker + retry khi gọi payment gateway bên thứ ba là pattern gần như bắt buộc với mọi sản phẩm có thanh toán online. |
| `client-storage-ecommerce-cart-draft-persistence` | Lưu giỏ hàng/form dở dang qua localStorage, merge guest-login, đồng bộ đa tab là bài toán frontend rất thường gặp. |
| `deadlock-ecommerce-order-inventory-coupon` | Deadlock giữa order/inventory/coupon lúc checkout là tình huống rất thường gặp ở bất kỳ sàn ecommerce có traffic thật nào. |
| `distributed-lock-cache-thundering-herd` | Chống thundering herd bằng lock/single-flight khi cache miss là vấn đề rất thường gặp ở hệ thống có traffic thật. |
| `distributed-lock-cron-job-scaled-instances` | "Cron chạy đúng 1 lần khi service scale nhiều instance" cực phổ biến, gần như service nào chạy nhiều replica cũng gặp. |
| `graceful-shutdown-api-gateway-rolling-restart` | Rolling restart không downtime ở tầng gateway là thực hành DevOps/backend rất phổ biến trong môi trường cloud-native. |
| `locking-crm-customer-profile-concurrent-edit` | Field-level optimistic lock cho hồ sơ khách hàng là tình huống rất thường gặp ở các sản phẩm CRM/SaaS B2B. |
| `multi-tenancy-project-management-shared-schema` | Mô hình shared-schema + tenant_id là kiểu multi-tenancy phổ biến nhất, gặp ở hầu hết mọi SaaS B2B. |
| `payment-gateway-webhook-out-of-order` | Hầu như mọi công ty tích hợp Stripe/VNPay đều phải xử lý webhook idempotent và out-of-order. |
| `reconciliation-ecommerce-payment-gateway-orders` | Đối soát định kỳ giữa cổng thanh toán và đơn hàng là công việc rất phổ biến ở ecommerce/SaaS nhận thanh toán online. |
| `schema-migration-ecommerce-orders-split` | Tách cột JSON sang bảng quan hệ không downtime là pattern schema migration rất phổ biến. |
| `search-indexing-ecommerce-product-search` | Tìm kiếm/autocomplete cho catalog ecommerce là nhu cầu rất phổ biến ở công ty bán hàng online quy mô kha khá. |

## Điểm 7

| Đề bài | Lý do |
|---|---|
| `blue-green-deployment-saas-core-service` | Blue-green deploy là kỹ thuật vận hành phổ biến ở SaaS có traffic thật deploy thường xuyên. |
| `cache-invalidation-cms-multilayer` | Invalidate cache nhiều tầng (CDN+app+query) là vấn đề thực tế với bất kỳ sản phẩm nào chạy sau CDN. |
| `canary-deployment-mobile-backend-api-versioning` | Backend phải serving song song nhiều phiên bản app mobile cũ là tình huống rất thường gặp ở công ty có app di động thực tế. |
| `client-storage-social-feed-scroll-multi-tab` | Khôi phục scroll infinite-feed qua bfcache, đồng bộ like đa tab là vấn đề quen thuộc với mọi sản phẩm dạng feed. |
| `deadlock-digital-bank-internal-transfer` | Deadlock khi chuyển tiền 2 chiều + lock ordering là pattern RDBMS phổ biến, gặp ở bất kỳ hệ thống nào có ví/số dư. |
| `distributed-lock-worker-pool-job-distribution` | Worker pool claim job bằng lease/TTL là pattern rất phổ biến trong các hệ thống dùng queue. |
| `distributed-tracing-mobile-backend-thirdparty` | Backend mobile gọi ra bên thứ ba (SMS OTP, map, push) là tình huống rất thường gặp ở công ty làm app di động. |
| `graceful-shutdown-ecommerce-checkout-deploy` | Deploy checkout service nhiều lần/ngày mà không cắt ngang giao dịch là vấn đề thực tế thường gặp ở ecommerce có traffic. |
| `inventory-reservation-ecommerce-multi-warehouse` | Soft-reserve tồn kho đa kho khi vào checkout là pattern phổ biến ở ecommerce có quy mô/nhiều kho thật. |
| `inventory-reservation-fashion-size-color-variant` | Tồn kho theo biến thể (size/màu) rất phổ biến trong ecommerce nói chung, không chỉ ngành thời trang. |
| `load-balancing-saas-b2b-tenant-weighted-routing` | Vấn đề "noisy neighbor" per-tenant rất thực tế ở các SaaS B2B có nhiều khách hàng lớn nhỏ khác nhau. |
| `locking-internal-config-multi-engineer` | Dashboard config/feature flag nội bộ nhiều kỹ sư cùng sửa là tình huống phổ biến ở công ty quy mô kỹ thuật kha khá. |
| `mfa-b2b-saas-org-policy` | Org admin tự cấu hình chính sách MFA là tính năng thường thấy ở các SaaS B2B có settings bảo mật cho doanh nghiệp. |
| `multi-tenancy-marketplace-seller-isolation` | Cách ly dữ liệu bán hàng giữa các seller chỉ chắc gặp nếu làm đúng loại hình sản phẩm marketplace. |
| `rate-limiting-saas-api-gateway-per-key` | Rate limit theo API key/gói dịch vụ là bài toán chuẩn của các SaaS cung cấp public API. |
| `saga-ecommerce-checkout` | Saga orchestration cho checkout đa service (payment/inventory/shipping) là pattern kinh điển ở ecommerce microservices. |
| `schema-migration-social-posts-comments-split` | Tách JSON comment lồng nhau sang bảng quan hệ dưới traffic cao khá tổng quát, gặp ở nhiều app có bình luận. |

## Điểm 6

| Đề bài | Lý do |
|---|---|
| `cache-invalidation-flash-sale-price-inventory` | Chỉ công ty ecommerce có flash sale với tần suất thay đổi cực cao mới cần độ khắt khe này. |
| `cache-invalidation-session-permission` | Cần đúng loại SaaS B2B có hệ thống RBAC/permission phức tạp cache lại để tối ưu. |
| `canary-deployment-ecommerce-checkout-highstakes` | Canary riêng cho checkout với rollback theo giây chỉ cần thiết ở ecommerce traffic lớn. |
| `circuit-breaker-llm-provider-chat-ai` | Ngày càng phổ biến khi nhiều SaaS thêm tính năng AI, nhưng vẫn cần đúng loại sản phẩm có tích hợp AI. |
| `client-storage-cms-editor-autosave-draft` | Autosave draft bằng IndexedDB thực tế với sản phẩm có trình soạn thảo nội dung dài (CMS, docs tool). |
| `client-storage-fintech-secure-session-token` | Pattern lưu token an toàn chỉ thực sự cần ở sản phẩm nhạy cảm bảo mật (fintech, SaaS enterprise). |
| `deadlock-game-leaderboard-update` | Deadlock giữa job tính rank hàng loạt và update điểm real-time gặp ở app có leaderboard/gamification. |
| `deadlock-hotel-price-inventory-update` | Chỉ chắc chắn gặp ở đúng loại hình sản phẩm đặt phòng/du lịch có tồn kho theo ngày. |
| `deadlock-warehouse-bidirectional-transfer` | Bài toán quen thuộc ở công ty có hệ quản lý kho/logistics, ít phổ biến ngoài domain đó. |
| `device-trust-marketplace-anomaly-stepup` | Phát hiện session bất thường + step-up trước thanh toán là pattern chống gian lận khá phổ biến ở sàn TMĐT. |
| `distributed-lock-flash-sale-inventory-reservation` | Chỉ gặp rõ ràng ở công ty ecommerce có traffic đột biến thật sự. |
| `distributed-tracing-b2b-saas-request` | Phổ biến ở công ty có kiến trúc microservices thật, không phải mặc định mọi nơi. |
| `distributed-tracing-ecommerce-order-flow` | Cần ecommerce có nhiều service tách biệt (cart/inventory/payment) mới thực sự gặp. |
| `graceful-shutdown-chat-server-realtime` | Chỉ áp dụng rõ với sản phẩm có tính năng chat/real-time WebSocket. |
| `inventory-reservation-c2c-marketplace` | Xung đột buyer/seller trên tồn kho tự quản chỉ rõ ràng ở mô hình marketplace C2C. |
| `load-balancing-ecommerce-flash-sale-spike` | Traffic spike, outlier detection là vấn đề thực tế ở ecommerce có traffic thật nhưng cần đúng quy mô. |
| `locking-project-management-gantt-collaborative` | Task-level optimistic lock chỉ chắc gặp nếu sản phẩm đúng là công cụ quản lý dự án dạng Asana/Jira. |
| `multi-tenancy-internal-tool-department-isolation` | Cách ly dữ liệu theo phòng ban nội bộ khá phổ biến ở công ty vừa/lớn có internal tool. |
| `payment-saas-subscription-renewal-race` | Race giữa job renewal tự động và đổi/hủy gói khá thường gặp ở SaaS B2B tự xây billing. |
| `rate-limiting-chat-spam-prevention` | Gặp khi sản phẩm có tính năng chat/nhắn tin ở quy mô kha khá, không phải mọi app đều có. |
| `rate-limiting-external-partner-sms-gateway` | Throttle khi gọi API đối tác ngoài (SMS/OTP) khá phổ biến với công ty gửi OTP/thông báo quy mô vừa. |
| `rate-limiting-flash-sale-checkout-throttle` | Virtual waiting room cho flash sale chỉ cần thiết ở ecommerce có traffic cao đột biến theo sự kiện. |
| `reconciliation-saas-subscription-billing-processor` | Đối soát billing nội bộ với payment processor khá phổ biến ở SaaS thuê bao quy mô kha khá. |
| `schema-migration-logistics-shipment-status-history` | Pattern khá tổng quát nhưng khung cảnh multi-partner logistics thu hẹp phạm vi gặp. |
| `search-indexing-job-board-listing-freshness` | Freshness index cho tin đăng quen thuộc ở nền tảng listing nhưng cần đúng loại sản phẩm. |
| `search-indexing-marketplace-multi-category-facet` | Facet động đa category chỉ cần thiết ở marketplace đủ lớn có nhiều ngành hàng khác biệt. |
| `search-indexing-saas-help-center-semantic` | Semantic search cho help center ngày càng phổ biến ở SaaS B2B có tài liệu lớn. |
| `service-discovery-saas-microservices-registry` | Service registry cơ bản là nhu cầu thường gặp ở công ty đã chia nhỏ thành nhiều microservice. |
| `sso-b2b-saas-okta-azure-ad` | Tích hợp SAML/OIDC với IdP khách hàng là yêu cầu chuẩn khi bán SaaS cho doanh nghiệp. |

## Điểm 5

| Đề bài | Lý do |
|---|---|
| `account-recovery-b2b-saas-sso-idp-lost` | Chỉ gặp nếu làm SaaS B2B có khách hàng enterprise dùng SSO/IdP riêng. |
| `cache-invalidation-social-feed-profile` | Fan-out invalidate feed cho follower quy mô lớn chỉ xảy ra ở sản phẩm mạng xã hội có traffic đủ lớn. |
| `circuit-breaker-shipping-carrier-api` | Circuit breaker riêng theo từng carrier chỉ cần thiết ở công ty logistics/ecommerce tích hợp nhiều đối tác giao hàng. |
| `client-storage-saas-offline-indexeddb-sync` | Kỹ thuật khá nâng cao, chỉ cần khi SaaS thực sự yêu cầu hoạt động offline. |
| `flash-sale-event-ticketing-seat-selection` | Giữ ghế theo sơ đồ cụ thể chỉ phổ biến ở nền tảng ticketing/booking. |
| `payment-ewallet-balance-concurrent-topup-payment` | Pattern atomic update số dư khá phổ biến nhưng chỉ rõ nét khi sản phẩm có tính năng ví/số dư nội bộ. |
| `reconciliation-marketplace-seller-payout` | Đối soát payout cho seller chỉ xuất hiện ở mô hình marketplace đa người bán cụ thể. |
| `saga-subscription-plan-change` | Đồng bộ billing-entitlement-notification khi đổi gói cần SaaS B2B có hệ thống entitlement riêng. |
| `schema-migration-fintech-currency-cents` | Thực hành tốt phổ biến khi xử lý tiền, nhưng dự án migrate quy mô lớn chỉ gặp khi hệ thống đã tồn tại lỗi này. |
| `search-indexing-real-estate-map-autocomplete` | Tìm kiếm theo bản đồ/geo autocomplete chỉ gặp ở sản phẩm có yếu tố địa lý mạnh. |
| `sso-enterprise-internal-portal` | Xây IdP nội bộ cho nhiều app công ty là nhu cầu có thật, nhưng nhiều nơi mua sẵn Okta thay vì tự xây. |
| `video-transcoding-youtube-like-sharing` | Bài toán VOD kinh điển nhưng đa số công ty dùng dịch vụ thứ ba (Mux, MediaConvert) thay vì tự xây. |

## Điểm 4

| Đề bài | Lý do |
|---|---|
| `account-recovery-enterprise-break-glass` | Quy trình break-glass/helpdesk-only reset đặc thù hệ thống nội bộ doanh nghiệp bảo mật cao. |
| `account-recovery-mobile-sim-swap` | Chống SIM-swap chỉ quan trọng với app dùng SĐT làm định danh chính (fintech/super-app). |
| `canary-deployment-ml-model-replacement` | Canary theo business metric cho ML serving là vấn đề đặc thù vai trò MLE/recsys. |
| `cap-theorem-flash-sale-quorum-counter` | Tự cấu hình quorum W/R linh hoạt là bài toán distributed-systems nâng cao, chỉ gặp ở ecommerce quy mô rất lớn. |
| `cap-theorem-payment-session-consistency` | Tunable CP/AP theo từng API là thiết kế distributed-systems nâng cao, ít công ty tự làm ở mức chi tiết này. |
| `cap-theorem-shopping-cart-kv-store` | Tự cấu hình N/W/R cho KV store phân tán là việc của đội infra/platform, không phải dev ứng dụng phổ thông. |
| `cap-theorem-social-follow-follower-counter` | Cấu hình quorum/CRDT counter chỉ liên quan tới mạng xã hội quy mô lớn. |
| `consistent-hashing-chat-message-sharding` | Sharding theo conversation ở quy mô triệu hội thoại chỉ gặp ở đội hạ tầng của app chat lớn. |
| `consistent-hashing-distributed-rate-limiter` | Tự xây rate limiter sharded là việc của đội platform/infra, đa số dev chỉ dùng thư viện có sẵn. |
| `device-trust-zero-trust-b2b-saas` | Zero Trust dựa trên MDM thường mua từ vendor hơn tự xây, chỉ gặp ở B2B SaaS yêu cầu bảo mật cao. |
| `distributed-tracing-video-transcode-pipeline` | Pipeline xử lý video nhiều bước bất đồng bộ chỉ gặp ở công ty làm sản phẩm media/video cụ thể. |
| `flash-sale-sneaker-drop-anti-bot` | Anti-bot + limited drop là mô hình bán hàng khá đặc thù (sneaker/hàng hiếm). |
| `graceful-shutdown-video-streaming-server` | Drain kết nối streaming dài hạn chỉ đặc thù cho công ty tự vận hành server phát video. |
| `inventory-reservation-dropshipping-supplier` | Đồng bộ tồn kho với nhà cung cấp bên thứ ba là mô hình kinh doanh dropshipping khá đặc thù. |
| `inventory-reservation-fnb-shared-ingredient` | Trừ nguyên liệu dùng chung nhiều món chỉ đặc thù cho hệ thống order F&B/nhà hàng. |
| `locking-warehouse-stocktake-concurrent-transactions` | Kiểm kho song song với giao dịch kho là bài toán đặc thù ngành bán lẻ/logistics. |
| `mfa-devops-hardware-key-privileged` | WebAuthn/hardware key cho tài khoản đặc quyền production là việc của vai trò platform/security engineering. |
| `multi-tenancy-payroll-saas-isolation` | Payroll/HR SaaS với hai lớp cách ly là một ngách sản phẩm khá hẹp. |
| `oauth-bff-mobile-secure` | Pattern BFF+PKCE khá tổng quát nhưng bài toán cụ thể (Open Banking liên kết nhiều ngân hàng) gắn chặt ngành tài chính. |
| `saga-marketplace-seller-payout` | Kết hợp cả tính đặc thù marketplace lẫn độ phức tạp orchestration. |
| `schema-migration-b2b-saas-schema-per-tenant` | Chuyển shared schema sang schema-per-tenant là bài toán scale nâng cao, chỉ gặp ở SaaS B2B đã lớn. |
| `search-indexing-social-user-hashtag-post` | Trending hashtag/personalized autocomplete chỉ có ý nghĩa ở mạng xã hội, một vertical khá hiếm. |
| `service-discovery-payment-multi-region` | Cross-region failover cho service discovery là bài toán hạ tầng quy mô lớn. |
| `service-discovery-service-mesh-sidecar` | Tự viết logic sidecar/mesh chỉ có ở đội platform/infra vì đa số công ty dùng sẵn Istio/Envoy. |
| `video-transcoding-elearning-subtitle-chapter-sync` | Đồng bộ phụ đề/chapter chỉ cần thiết ở nền tảng elearning có pipeline transcode riêng. |
| `video-transcoding-livestream-vod-postprocessing` | Hậu xử lý VOD từ segment livestream chỉ gặp ở nền tảng streaming trực tiếp. |
| `video-transcoding-marketplace-seller-listing-video` | Chuẩn hóa video seller tự upload chỉ cần thiết ở marketplace có tính năng video sản phẩm. |
| `video-transcoding-short-video-tiktok-like` | Pipeline transcode siêu nhanh cho video ngắn chỉ thực sự cần ở app video ngắn kiểu TikTok. |
| `wal-order-processing-in-memory-cache` | Áp dụng nguyên lý WAL bảo vệ cache đơn hàng là pattern ứng dụng cụ thể hơn nhưng vẫn ít gặp với dev thông thường. |

## Điểm 3

| Đề bài | Lý do |
|---|---|
| `account-recovery-fintech-lost-channels` | Gắn chặt với ví điện tử/đầu tư có tiền thật và compliance tài chính nghiêm ngặt (KYC, audit trail). |
| `cap-theorem-infra-metrics-dashboard-alerting` | Tinh chỉnh consistency cho cụm time-series phân tán là công việc của đội platform/infra chuyên biệt. |
| `circuit-breaker-ridehailing-maps-routing` | Circuit breaker cho API bản đồ real-time rất đặc thù của ride-hailing/delivery. |
| `consistent-hashing-largescale-session-store` | Session store cho hàng chục triệu user online đồng thời chỉ tồn tại ở quy mô nền tảng cực lớn. |
| `consistent-hashing-redis-cluster-resharding` | Thiết kế/resharding hệ cache kiểu Redis Cluster từ đầu là việc của đội xây hạ tầng cache. |
| `consistent-hashing-time-series-metrics` | Xây hệ lưu trữ time-series metrics tự phân vùng chỉ thuộc đội observability/infra chuyên biệt. |
| `device-trust-fintech-highvalue-transfer` | Mô hình "device maturity" trước giao dịch giá trị lớn là yêu cầu bảo mật đặc thù ngân hàng số. |
| `device-trust-messaging-linked-devices` | Mô hình thiết bị chính + liên kết (kiểu WhatsApp Web) chỉ liên quan nếu trực tiếp xây sản phẩm messaging đa thiết bị. |
| `device-trust-streaming-device-slot-limit` | Giới hạn slot thiết bị xem đồng thời đặc thù mô hình kinh doanh streaming thuê bao. |
| `distributed-lock-multi-region-deploy-coordination` | Điều phối deploy cross-region chỉ có ở hạ tầng multi-region quy mô lớn. |
| `distributed-tracing-fintech-audit-observability` | Audit trail bất biến, retention dài theo compliance gắn chặt với ví điện tử/thanh toán được quản lý. |
| `flash-sale-digital-bank-loan-offer` | Mở suất vay ưu đãi kiểm tra credit bureau là nghiệp vụ đặc thù ngân hàng số/cho vay. |
| `live-streaming-live-shopping-checkout-sync` | Đồng bộ chốt đơn với mốc thời gian trên luồng live là bài toán ngách của nền tảng live-commerce. |
| `live-streaming-online-classroom-interactive-sync` | Kỹ thuật ingest/sync video chuyên biệt của ngành streaming/edtech. |
| `live-streaming-webinar-multi-presenter-mixing` | Xây engine mixing/switch nhiều luồng video real-time là việc của đội platform media chuyên biệt (kiểu Zoom). |
| `load-balancing-cdn-video-streaming-edge` | Load balancing tại edge CDN là bài toán hạ tầng chuyên biệt chỉ công ty streaming/CDN lớn mới tự làm. |
| `load-balancing-fintech-session-affinity` | Tự implement consistent hashing/session affinity cho giao dịch tài chính là việc của đội infra fintech. |
| `load-balancing-game-matchmaking-server` | Consistent hashing theo room cho game server chỉ xuất hiện ở ngành game multiplayer. |
| `mfa-digital-bank-mandatory` | MFA/step-up cho giao dịch ngân hàng số gắn chặt với ngành ngân hàng chịu quy định pháp lý riêng. |
| `mfa-social-creator-account-takeover-fatigue` | Chính sách bảo vệ tài khoản creator/influencer chỉ có ý nghĩa ở số ít nền tảng mạng xã hội quy mô lớn. |
| `oauth-provider-third-party-ecosystem` | Tự xây Authorization Server cho hệ sinh thái app bên thứ ba chỉ cần ở công ty nền tảng kiểu Shopify. |
| `payment-cross-border-multi-timezone-callback` | Khóa tỷ giá xuyên biên giới + đối soát đa múi giờ rất đặc thù của công ty thanh toán quốc tế. |
| `payment-orchestration-multi-provider-routing` | Định tuyến qua nhiều payment provider chỉ có ở công ty thanh toán/marketplace quy mô lớn. |
| `reconciliation-ride-hailing-trip-driver-payout` | Đối soát cuốc xe/hoa hồng/tiền mặt tài xế đặc thù riêng cho ngành gọi xe. |
| `saga-ride-hailing-trip-lifecycle` | Điều phối vòng đời chuyến đi qua nhiều thiết bị/service đặc thù riêng ngành ride-hailing. |
| `saga-travel-combo-booking` | Đặt combo đa nhà cung cấp (vé máy bay+khách sạn+xe) chỉ đặc thù cho nền tảng du lịch/OTA. |
| `service-discovery-retail-pos-intermittent-connectivity` | Quản lý fleet thiết bị POS với kết nối gián đoạn là vấn đề đặc thù ngành bán lẻ có hạ tầng thiết bị riêng. |
| `sso-consumer-platform-multi-app-ecosystem` | Chỉ công ty vận hành nhiều app tiêu dùng lớn dùng chung 1 danh tính trung tâm (quy mô kiểu Google) mới cần. |
| `sso-fintech-embedded-finance` | SSO xuyên web/mobile/đối tác nhúng cho embedded finance khá đặc thù của mảng fintech/BaaS. |
| `video-transcoding-saas-b2b-webinar-fast-delivery` | Xử lý bản ghi webinar tốc độ cao kèm tách speaker theo track là nhu cầu hẹp của SaaS họp trực tuyến. |

## Điểm 2 — Cực hiếm (ngành hạn chế giấy phép hoặc hạ tầng lõi)

| Đề bài | Lý do |
|---|---|
| `account-recovery-healthcare-urgent-verification` | Dữ liệu y tế nhạy cảm, quy định bảo vệ dữ liệu sức khỏe và luồng emergency-access là đặc thù ngành y tế được cấp phép. |
| `circuit-breaker-digital-bank-legacy-core` | Bảo vệ hệ thống core banking legacy là bài toán đặc thù ngành ngân hàng số. |
| `consensus-realtime-game-server-cluster` | Tự xây consensus cho cụm game server authoritative chỉ đội infra/network engine của studio game lớn mới làm. |
| `deployment-core-banking-audit-controlled` | Canary deployment kèm four-eyes approval và audit trail bất biến chỉ bắt buộc ở hệ thống ngân hàng lõi bị quản lý chặt. |
| `live-streaming-entertainment-twitch-like` | Tự xây hạ tầng ingest/transcode/CDN cho streaming là công việc hạ tầng chuyên biệt, đa số dev dùng dịch vụ bên thứ ba. |
| `live-streaming-gaming-multi-region-ingest` | Định tuyến ingest đa vùng toàn cầu chỉ vài công ty (Twitch-scale) cần tới. |
| `live-streaming-sports-betting` | Kết hợp ngành cá cược bị hạn chế giấy phép với hạ tầng streaming latency-đồng nhất cực đặc thù. |
| `mfa-telehealth-risk-based` | Adaptive MFA gắn với tuân thủ dữ liệu y tế là bài toán của ngành y tế được cấp phép. |
| `multi-tenancy-healthcare-strict-isolation` | Cách ly dữ liệu bệnh nhân với compliance y tế nghiêm ngặt (crypto shredding, audit theo tenant). |
| `reconciliation-digital-bank-card-network` | Đối soát ngân hàng số với mạng thẻ kèm audit trail kiểm toán là đặc thù ngành ngân hàng lõi được cấp phép. |
| `sso-healthcare-hospital-clinic-network` | Gắn chặt với ngành y tế được cấp phép, yêu cầu audit truy cập hồ sơ bệnh nhân và break-glass đặc thù. |
| `wal-digital-bank-transaction-log` | Gắn với ngân hàng số/core banking, yêu cầu durability đa vị trí và audit đối soát đặc thù ngành tài chính. |
| `wal-distributed-kv-store-node` | Xây node lưu trữ phân tán tự phối hợp WAL với Raft log ở tầng cluster là công việc hạ tầng cấp thấp hiếm gặp. |
| `wal-message-queue-broker` | Tự xây message broker đảm bảo durability bằng WAL là hạ tầng lõi mà đa số công ty dùng Kafka/RabbitMQ có sẵn. |

## Điểm 1 — Gần như không ai gặp

| Đề bài | Lý do |
|---|---|
| `consensus-config-store-etcd-like` | Tự cài Raft từ đầu cho config store — công việc hạ tầng lõi cực hiếm. |
| `consensus-custom-message-broker-ordering` | Tự xây message broker với consensus đảm bảo thứ tự — công việc hạ tầng lõi cực chuyên biệt. |
| `consensus-distributed-sql-sharding` | Tự xây distributed SQL DB kiểu CockroachDB với multi-Raft — bản chất là viết database engine. |
| `consensus-financial-ledger-linearizability` | Kết hợp cả ledger ngân hàng lẫn tự cài consensus/Raft từ đầu — vừa ngành hạn chế vừa hạ tầng lõi. |
| `reconciliation-crypto-onchain-ledger` | Đối soát ví on-chain với ledger nội bộ, proof-of-reserves chỉ tồn tại ở sàn giao dịch crypto. |
| `wal-custom-database-engine` | Tự viết storage engine + WAL từ đầu là công việc hạ tầng lõi cực hiếm. |
