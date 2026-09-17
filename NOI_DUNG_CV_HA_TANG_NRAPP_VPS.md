# Nội dung CV về hạ tầng NRApp đã xác minh trên VPS

Kiểm tra trực tiếp ngày 16/09/2026, khoảng 18:42 giờ Việt Nam, trên VPS
`kien24` (`103.116.52.35`). Source được đọc để hiểu thiết kế; các công nghệ và
con số trong phần CV chỉ lấy từ cấu hình, container, API và kết quả thực thi
trên môi trường đã triển khai. Đây là mô tả thành phần đang vận hành, không phải
cam kết hiệu năng hoặc số lượng người dùng production.

## Ba ý có thể dùng trong CV

1. Triển khai giám sát tài nguyên VPS bằng Prometheus, Grafana và node_exporter,
   thu thập metrics CPU, RAM và dung lượng đĩa mỗi 60 giây; vận hành healthcheck
   cho các container và ghi log JSON kèm request_id để hỗ trợ truy vết yêu cầu.
2. Đóng gói và triển khai 8 microservices Node.js 22/NestJS 11 trên VPS Ubuntu
   22.04 với 2 vCPU, khoảng 4 GB RAM bằng Docker Compose và Nginx HTTPS; xây dựng
   CI/CD theo từng service bằng GitHub Actions, kiểm tra sức khỏe khi triển khai,
   khóa tránh deploy đồng thời và cơ chế rollback image khi rollout thất bại.
3. Tự động sao lưu MongoDB Atlas và Redis hằng ngày lúc 02:30 bằng systemd timer,
   tạo bản backup theo timestamp, mã hóa bằng Age và kiểm tra SHA-256; lưu trên
   VPS theo chính sách 14 ngày, tối thiểu 7 bản, và trên GitHub Actions artifact
   30 ngày; diễn tập khôi phục MongoDB thành công với 19 collections và 53 indexes.

## Số liệu và công nghệ thực tế

| Nội dung | Kết quả xác minh | Bằng chứng |
|---|---|---|
| Hệ điều hành | Ubuntu 22.04.5 LTS | `lsb_release -ds` trên VPS |
| CPU/RAM | 2 CPU khả dụng; RAM hệ thống báo 3911 MiB | `nproc`, `free -m`; CV ghi khoảng 4 GB RAM |
| Service ứng dụng | 8 container, đều healthy khi kiểm tra | `docker ps`, mỗi service một container |
| Hạ tầng/monitoring | Redis, RabbitMQ, Prometheus, Grafana, node_exporter | 5 container healthy; tổng cùng ứng dụng là 13 |
| Node.js | v22.23.2 trên cả 8 service | `process.version` trong từng container |
| NestJS | 11.x, bản cụ thể theo bảng bên dưới | `@nestjs/core/package.json` trong container |
| Prometheus | 3.12.0 | API `/api/v1/status/buildinfo` |
| Grafana | 13.2.0, database `ok` | API `/api/health` trên VPS |
| node_exporter | Image v1.11.1 | Image container đang chạy |
| Scrape interval | `1m`, tức 60 giây | Cấu hình đang nạp qua `/api/v1/status/config` |
| Prometheus targets | 2 target UP: `node-exporter`, `prometheus` | API `/api/v1/targets` |
| Metrics của VPS | Có CPU, RAM khả dụng, dung lượng filesystem | Query `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes` |
| Grafana datasource | Prometheus, `http://prometheus:9090` | File provisioning trong container Grafana |
| Logging | Cả 8 service cấu hình `LOG_FORMAT=json`; mẫu log Gateway có `request_id` | Docker env được lọc và mẫu log JSON, không xuất nội dung nhạy cảm |
| Metrics/tracing ứng dụng | OTel đang tắt: `OTEL_SDK_DISABLED=true`, `OTEL_METRICS_EXPORTER=none` | Env thực tế trong cả 8 container |
| Deployment | Receiver có deployment lock, healthcheck wait và rollback logic | `/home/deploy/bin/nrapp-ci-receiver` trên VPS |
| Backup service | `Result=success`, `ExecMainStatus=0` | `systemctl --user show nrapp-backup.service` |
| Backup timer | 02:30 Việt Nam, trễ ngẫu nhiên tối đa 5 phút, persistent | Unit đang cài: `19:30:00 UTC`, `RandomizedDelaySec=300` |
| Backup trên VPS | 2 gói mã hóa khoảng 31 KB, tạo lúc 16:02 và 16:03 ngày kiểm tra | Các file thực tế trong `/opt/nrapp-backups` |
| Lưu ngoài VPS | GitHub artifact 31357 bytes, chưa hết hạn; hạn đến 16/10/2026 | GitHub Actions API và workflow thành công |
| Lưu bản sao trên máy cá nhân | Đã tải bản mã hóa từ GitHub để kiểm tra | `/home/thanhle/.config/nrapp-backup/verification/encrypted` trong lần diễn tập |

Số lượng target, container, collections và dung lượng backup là số liệu tại
thời điểm kiểm tra; không phải giới hạn cố định của hệ thống.

## Phân biệt 9 service repo và 8 service đang chạy

| Service | Trạng thái trên VPS | Node.js | NestJS core |
|---|---|---|---|
| Gateway | Healthy | 22.23.2 | 11.1.28 |
| Auth | Healthy | 22.23.2 | 11.1.27 |
| User | Healthy | 22.23.2 | 11.2.1 |
| Mail | Healthy | 22.23.2 | 11.2.1 |
| Chat | Healthy | 22.23.2 | 11.2.1 |
| Todo | Healthy | 22.23.2 | 11.2.1 |
| Workschedule | Healthy | 22.23.2 | 11.2.1 |
| Canteen | Healthy | 22.23.2 | 11.1.27 |
| Payment | Không có container chạy trong lần kiểm tra | Không đưa phiên bản local vào số liệu VPS | Không đưa phiên bản local vào số liệu VPS |

Logger là repository chứa package observability và workflow dùng chung;
nrapp-backup là repository chứa mã backup. Không cộng hai repo này thành
microservice ứng dụng đang chạy trên VPS.

## Những ý trong bản mẫu cần sửa

| Ý trong bản mẫu | Kết quả đối chiếu môi trường triển khai |
|---|---|
| PM2 monitoring | Không tìm thấy PM2 trên PATH của user deploy hoặc dependency PM2 trong 8 container; môi trường kiểm tra dùng Docker |
| Circuit breaker | Chưa có bằng chứng circuit breaker đã triển khai; source có HTTP timeout, không đủ để gọi là circuit breaker |
| Cảnh báo Discord/Telegram | Source có cấu hình/example, nhưng chưa xác nhận delivery trên VPS; Prometheus đang có 0 rules và 0 Alertmanager active |
| Theo dõi độ trễ service bằng Prometheus | Chưa xác nhận trên VPS: hiện chỉ có 2 target hạ tầng và OTel ứng dụng đang tắt |
| Phát hiện 25 sự cố production | Chưa có hồ sơ incident hoặc thống kê đã xác nhận để dùng con số này |
| Cụm 3 node | Môi trường kiểm tra là một VPS Docker Compose; Swarm inactive, chưa có bằng chứng cụm 3 node |
| Load balancing và horizontal scaling | Mỗi service đang có một container; Nginx chuyển tiếp API về một Gateway `127.0.0.1:3000`, chưa chứng minh phân phối qua nhiều replica/node |
| MySQL | Không có container MySQL đang chạy hoặc thành phần backup MySQL trong hệ thống này |
| PostgreSQL | Payment/PostgreSQL chưa chạy; script backup chỉ có nhánh hỗ trợ khi container chạy, chưa coi là backup PostgreSQL đã vận hành |
| Cron 04:00 | Thực tế dùng systemd timer 02:30; GitHub schedule lấy bản sao lúc 03:15 giờ Việt Nam |
| Tự động khôi phục mỗi ngày | Hiện tự động backup; restore MongoDB đã được diễn tập thủ công vào container riêng |
| Lưu trữ 7 ngày | Chính sách thực tế là VPS 14 ngày, tối thiểu 7 bản; GitHub 30 ngày. “7 bản” khác “7 ngày” |
| Google Drive | Chưa có tích hợp trong mã backup đang triển khai và chưa có bằng chứng bản sao trên Drive |

Không dùng số alert rules, số log lỗi hoặc số test thay cho số sự cố production
đã phát hiện. Cũng chưa có số liệu load test để ghi throughput, p95/p99, mức giảm
độ trễ, tỷ lệ uptime, khả năng mở rộng hoặc RTO đã đạt.

## Phạm vi và kết quả backup/restore đã kiểm chứng

Backup gồm MongoDB database `nrapp`, Redis RDB, RabbitMQ definitions và cấu hình
backend. RabbitMQ definitions không chứa message đang chờ. Không có file media
Cloudinary hoặc snapshot toàn bộ VPS. MongoDB là live dump từng collection,
không phải snapshot nhất quán toàn hệ thống trong lúc có thao tác ghi.

Lần diễn tập trước đó đã tải gói từ GitHub, xác minh checksum, giải mã trên máy
cá nhân và restore vào MongoDB riêng trên VPS: **19 collections, 4 documents,
53 indexes**, `mongorestore` báo **0 documents failed**. Đây là lượng dữ liệu thử
nhỏ; không dùng kết quả này để suy ra hiệu năng phục hồi database lớn hay mức tải
production. Container và dữ liệu thử đã được dọn. Redis đã qua `redis-check-rdb`,
nhưng chưa diễn tập phục hồi toàn bộ Redis/RabbitMQ/Payment/ứng dụng.

Bản sao trên máy cá nhân là kết quả tải về kiểm tra; chưa có lịch tự động kéo
backup về máy cá nhân. “Local” tự động trong phần CV là thư mục lưu trên VPS.

## Liên kết bằng chứng

- [Hướng dẫn backup và kết quả diễn tập](./HUONG_DAN_BACKUP_DB_VPS_GITHUB.md)
- [Backup ngoài VPS thành công](https://github.com/lethanh2006/nrapp-backup/actions/runs/35077097126)
- [CI/CD backup thành công](https://github.com/lethanh2006/nrapp-backup/actions/runs/35077360905)
- [CD Auth thành công](https://github.com/lethanh2006/AUTH_SERVICE/actions/runs/35057742975)

Các file source để hiểu thiết kế gồm `backend/logger/packages/observability`,
`backend/logger/compose.vps-minimal.yaml`, `backend/gateway/src/common/http`,
`backend/logger/.github/workflows` và `nrapp-backup/scripts/backup.py`. Những thành
phần có trong source nhưng không được xác nhận ở runtime không được đưa vào
ba ý CV như một công nghệ đã vận hành trên VPS.
