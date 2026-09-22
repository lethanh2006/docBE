# Mô tả tổng quan và kiến trúc NRApp trên product VPS

Phạm vi của tài liệu này là trạng thái đang chạy trên product VPS, được đối
chiếu với source và runtime ngày 21/09/2026. Các con số dưới đây là snapshot
của môi trường triển khai, không phải cam kết về tải, uptime hay số người dùng.

## Tổng quan

NRApp là nền tảng vận hành nội bộ gồm ứng dụng mobile Expo/React Native và API
backend theo kiến trúc microservices. Product backend hiện chạy 8 service
NestJS trên Node.js 22.23.2 bằng Docker Compose trên một VPS Ubuntu 22.04.5
(2 vCPU, khoảng 4 GB RAM, 50 GB disk): Gateway, Auth, User, Mail, Chat, Todo,
Workschedule và Canteen.

Các phân hệ đang được expose qua API gồm xác thực email/OTP và Google Sign-In,
quản lý tài khoản và vai trò, hồ sơ người dùng, công việc, lịch làm việc/chấm
công, chat realtime Socket.IO và vận hành căn tin. API production là:

- API/Swagger: <https://api-vps.thanhlelmtp2006.id.vn/api-docs>
- APK: <https://github.com/lethanh2006/Nrapp/releases/latest>
- Backend Gateway: <https://github.com/lethanh2006/API-GATEWAY>

## Kiến trúc triển khai

```text
Mobile app / web client
          |
          | HTTPS
          v
Nginx trên VPS (public 80/443)
          |
          v
API Gateway :3000
  |  JWT introspection, RBAC, validation, request-id, rate limit
  |  HTTP forwarding + Socket.IO upgrade
  +--> Auth        :4000  ---> MongoDB Atlas + Redis + RabbitMQ
  +--> User        :5000  ---> MongoDB Atlas + RabbitMQ consumer
  +--> Mail        :5001  ---> RabbitMQ consumer + SMTP
  +--> Chat        :5002  ---> MongoDB Atlas + Cloudinary
  +--> Todo        :5003  ---> MongoDB Atlas
  +--> Workschedule:5004  ---> MongoDB Atlas
  +--> Canteen     :5005  ---> MongoDB Atlas + Redis + RabbitMQ

RabbitMQ và Redis chạy trong Docker Compose;
MongoDB chạy ngoài VPS trên MongoDB Atlas, database `nrapp`.
```

Các port service chỉ bind vào loopback của VPS; Internet đi qua Nginx rồi tới
Gateway. Gateway kiểm tra access token bằng Auth Service, sau đó gửi identity
đã ký HMAC tới service nội bộ. Chat giữ REST API trong Chat Service và chuyển
tiếp kết nối Socket.IO/WebSocket qua Gateway.

## Các điểm kỹ thuật đã triển khai

- Mỗi service là một container production image `linux/amd64`, có healthcheck,
  restart policy và giới hạn memory. Tại thời điểm kiểm tra có 13 container:
  8 service ứng dụng, Redis, RabbitMQ, Prometheus, Grafana và node-exporter;
  tất cả container đang healthy.
- Auth dùng JWT HS256, refresh-token rotation và Redis cho OTP, giới hạn OTP
  và trạng thái phiên. Gateway thực hiện JWT introspection và RBAC; các request
  Gateway-to-service dùng header identity có chữ ký HMAC.
- Auth ghi credential và transactional outbox trong MongoDB rồi publish event
  bền vững qua RabbitMQ để đồng bộ User; luồng OTP được chuyển bất đồng bộ tới
  Mail Service. RabbitMQ có retry/dead-letter queues cho các consumer đã triển
  khai.
- Toàn bộ service dùng structured JSON log với `request_id`; Gateway tạo
  correlation ID, chuẩn hóa lỗi và áp dụng rate limit theo IP trong bộ nhớ của
  từng Gateway instance. Đây chưa phải rate limit dùng Redis dùng chung cho
  nhiều replica.
- CI/CD của từng service chạy quality gate, build image theo đúng commit,
  truyền image qua SSH restricted command tới VPS, chờ healthcheck và rollback
  image trước đó nếu rollout thất bại.
- Monitoring hiện là monitoring hạ tầng: Prometheus 3.12.0 scrape
  `node-exporter` và chính Prometheus mỗi 1 phút; Grafana 13.2.0 hiển thị
  metrics CPU, RAM và disk. Runtime đang có 2 target Prometheus ở trạng thái
  UP, chưa có alert rule/Alertmanager và chưa đủ bằng chứng để ghi application
  latency tracing.
- Backup chạy bằng systemd user timer khoảng 02:30 giờ Việt Nam, có random
  delay tối đa 5 phút và `Persistent=true`; archive MongoDB/Redis/RabbitMQ
  được mã hóa bằng Age, kiểm tra SHA-256 và lưu off-site qua GitHub Actions.

## Phạm vi chưa active trên product VPS

Payment Service vẫn có source cho VietQR/Casso và PostgreSQL, nhưng profile
Payment/PostgreSQL/pgAdmin không chạy trên VPS hiện tại. Nginx chủ động trả
`503 PAYMENT_TEMPORARILY_DISABLED` cho `/api/payment`; luồng căn tin đang dùng
thanh toán tiền mặt.

Vì vậy không nên mô tả hệ thống hiện tại là 14 service, cluster 3 node,
horizontal scaling/load balancing nhiều replica, PM2 monitoring, MySQL,
circuit breaker, Prometheus application tracing, Discord/Telegram alerting,
1000+ người dùng, uptime 24/7 trong 14 tháng hoặc throughput/p95/95% success
nếu không có một lần đo production riêng làm bằng chứng.
