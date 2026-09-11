# Kế hoạch thuê VPS và thực hành triển khai backend NRApp

Ngày lập: **11/09/2026**. Căn cứ: mã nguồn và cấu hình hiện có trong `backend/`, đã đọc [CLAUDE.md](../backend/CLAUDE.md).

**Phương án tiết kiệm được chọn:** một VPS **Linux Ubuntu 24.04 LTS x86_64, 2 vCPU/4 GB RAM**, Docker Compose chạy đủ **9 app + 3 hạ tầng + 3 giám sát = 15 container**. MongoDB Atlas đặt ngoài VPS; Nginx trên host làm cửa vào HTTPS. Giữ code/thư viện logger để dùng ở local; trên VPS chỉ chạy Prometheus, Grafana và Node Exporter. Bắt đầu với dữ liệu lab, tải thấp và một instance mỗi service.

**Cấu hình mua mục tiêu: 2 vCPU, 4 GB RAM, SSD/NVMe khoảng 50–60 GB, thêm 2 GB swap.** Ưu tiên build image ở máy cá nhân rồi chuyển lên VPS, không bật pgAdmin thường trực. Đây là cấu hình để thử nghiệm chi phí thấp, **chưa có benchmark chứng minh 9 app chạy ổn với giới hạn bên dưới**. Giữ đủ service, đo từng bước và điều chỉnh ngân sách RAM nếu có OOM; không cam kết số user hoặc khẳng định container nào chắc chắn bị kill trước.

Tài liệu này là kế hoạch và các mẫu để bạn thực hiện sau khi thuê VPS. **Việc tạo tài liệu chưa sửa source, chưa tạo VPS, chưa triển khai và chưa kiểm thử trên VPS thật.** Những cấu hình mới như `compose.vps.yaml` dưới đây là file bạn sẽ tạo trong quá trình thực hành.

## Mục lục

1. [Backend hiện có những gì](#1-backend-hiện-có-những-gì)
2. [Chọn VPS và dự trù chi phí](#2-chọn-vps-và-dự-trù-chi-phí)
3. [Chuẩn bị trước khi mua](#3-chuẩn-bị-trước-khi-mua)
4. [Tạo VPS, SSH và firewall](#4-tạo-vps-ssh-và-firewall)
5. [Cài Docker, Node và công cụ](#5-cài-docker-node-và-công-cụ)
6. [Đưa source lên VPS](#6-đưa-source-lên-vps)
7. [Chuẩn bị MongoDB và dịch vụ ngoài](#7-chuẩn-bị-mongodb-và-dịch-vụ-ngoài)
8. [Tạo env và đồng bộ secret](#8-tạo-env-và-đồng-bộ-secret)
9. [Cấu hình riêng cho VPS](#9-cấu-hình-riêng-cho-vps)
10. [Build và khởi động theo thứ tự](#10-build-và-khởi-động-theo-thứ-tự)
11. [DNS, Nginx và HTTPS](#11-dns-nginx-và-https)
12. [Kiểm tra từng luồng nghiệp vụ](#12-kiểm-tra-từng-luồng-nghiệp-vụ)
13. [Giám sát và cảnh báo](#13-giám-sát-và-cảnh-báo)
14. [Backup và diễn tập restore](#14-backup-và-diễn-tập-restore)
15. [Cập nhật, rollback và reboot](#15-cập-nhật-rollback-và-reboot)
16. [Tra cứu lỗi thường gặp](#16-tra-cứu-lỗi-thường-gặp)
17. [Lịch thực hành và tiêu chí hoàn thành](#17-lịch-thực-hành-và-tiêu-chí-hoàn-thành)
18. [Khi nào nâng cấp kiến trúc](#18-khi-nào-nâng-cấp-kiến-trúc)

## 1. Backend hiện có những gì

### 1.1. Chín service ứng dụng

| Service | Port container | Vai trò và phụ thuộc thấy trong source | Kiểm tra trên VPS |
|---|---:|---|---|
| `gateway` | 3000 | API công khai, JWT/role, ký request nội bộ, proxy HTTP và Socket.IO sang Chat | `/health`; chỉ phản ánh liveness |
| `auth` | 4000 | Credential MongoDB, OTP/refresh token Redis, gửi sự kiện RabbitMQ, đọc profile User | `/health`; chỉ phản ánh liveness |
| `user` | 5000 | Profile MongoDB, consumer `user-profile-sync` | `/health/ready` |
| `mail` | 5001 | Consumer `send-otp`, SMTP, retry và DLQ | `/health/ready` kiểm tra RabbitMQ và SMTP |
| `chat` | 5002 | MongoDB, Socket.IO, JWT, User API, upload Cloudinary | `/health/ready` |
| `todo` | 5003 | MongoDB và User API có chữ ký nội bộ | `/health/ready` |
| `workschedule` | 5004 | MongoDB, User API, lịch/duyệt yêu cầu/chấm công | `/health/ready` |
| `canteen` | 5005 | MongoDB, Redis, RabbitMQ, đơn hàng, kho, bếp, bàn, báo cáo | `/health/ready` kiểm tra MongoDB/Redis/RabbitMQ |
| `payment` | 5006 | PostgreSQL/TypeORM, VietQR, Casso V2, RabbitMQ/outbox | `/health/ready` |

`user`, `chat`, `todo`, `workschedule` có readiness MongoDB; không suy ra rằng tất cả API upstream và mọi nghiệp vụ đã hoạt động chỉ từ kết quả đó.

Các file đối chiếu chính:

- [Compose backend](../backend/compose.yaml), [Dockerfile dùng chung](../backend/docker/node-service.Dockerfile), [package.json gốc](../backend/package.json).
- [Gateway bootstrap](../backend/gateway/src/main.ts), [Auth forwarding đang mở trong IDE](../backend/gateway/src/modules/auth/auth.service.ts).
- [Auth nghiệp vụ](../backend/auth/src/modules/auth/auth.service.ts), [Chat socket](../backend/chat/src/modules/chat/chat.gateway.ts).
- [Payment database](../backend/payment/src/modules/database/database.options.ts), [Gateway payment routes](../backend/gateway/src/modules/payment/payment.controller.ts).

### 1.2. Hạ tầng và dữ liệu

| Thành phần | Hiện trạng | Ý nghĩa khi triển khai |
|---|---|---|
| Redis | Compose, AOF bật, volume `redis_data` | Lưu OTP, phiên refresh và dữ liệu Redis của ứng dụng; mất Redis có thể buộc đăng nhập lại |
| RabbitMQ | Compose, management UI, volume `rabbitmq_data` | Đưa mail, đồng bộ profile, sự kiện thanh toán qua queue |
| PostgreSQL | Service `payment-postgres`, volume `payment_postgres_data` | Dữ liệu Payment; host port mặc định **5433**, container port **5432** |
| MongoDB | **Chưa có trong Compose** | Tạo Atlas riêng cho lab hoặc tự dựng replica set ở giai đoạn sau |
| Cloudinary | Dịch vụ bên ngoài | Ảnh chat không nằm trong volume của VPS |
| SMTP | Dịch vụ bên ngoài | Đăng nhập email phụ thuộc gửi OTP thực sự thành công |
| pgAdmin | Profile `admin`, mặc định chưa chạy | Chỉ mở khi cần qua SSH tunnel |

MongoDB có điểm đặc biệt: Chat/Todo/Workschedule cố định `dbName: 'nrapp'`; Auth/User cho phép `MONGO_DB_NAME`, mặc định `nrapp`; Canteen lấy database theo URI. Vì vậy **dùng một Atlas cluster lab riêng, database `nrapp`, và đặt URI `/nrapp` nhất quán**. Đổi riêng URI sang `/nrapp_staging` chưa tách được toàn bộ service khỏi database cũ.

Canteen xuất kho và Workschedule duyệt lịch dùng `startSession()`/`withTransaction()`. MongoDB standalone không đáp ứng transaction nhiều document; chọn replica set/Atlas. Một replica set một node chỉ phục vụ bài học tự vận hành, không tạo khả năng chịu lỗi. [MongoDB: yêu cầu cho transactions](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/).

### 1.3. Cắt giám sát từ 13 xuống đúng 3 container

[Compose logger](../backend/logger/compose.yaml) chứa **13 container**: Jaeger, OTel Collector, Prometheus, Alertmanager, Node Exporter, cAdvisor, Blackbox Exporter, Redis Exporter, PostgreSQL Exporter, Loki, Alloy, Grafana.

VPS chỉ giữ **Prometheus + Grafana + Node Exporter**. Không chạy Jaeger, OTel Collector, Alertmanager, cAdvisor, Blackbox Exporter, Redis Exporter, PostgreSQL Exporter, Loki hoặc Alloy. Tổng mới **15 container**, giảm 10 container so với cấu hình đầy đủ 25. Giữ MongoDB ở Atlas, pgAdmin tắt mặc định; dùng công cụ DB trên máy cá nhân qua SSH tunnel khi cần.

**Không dùng** `node scripts/observability-compose.mjs up -d prometheus grafana node-exporter` để rút gọn: Compose gốc có `depends_on` kéo theo nhiều service khác, đồng thời Prometheus/Grafana còn trỏ tới các nguồn dữ liệu bị bỏ. Bước 9 tạo file **độc lập** `logger/compose.vps-minimal.yaml`; không merge file đó với `logger/compose.yaml`. [Docker: thứ tự và phụ thuộc khởi động](https://docs.docker.com/compose/how-tos/startup-order/).

Giữ structured log JSON ra stdout và đọc bằng `dc logs`; đặt `OTEL_SDK_DISABLED=true` trên đủ 9 app để không khởi động SDK/exporter tới Collector đã tắt. Thư viện vẫn được import theo code hiện tại nên không coi chi phí instrumentation đã về 0. Giới hạn `OBSERVABILITY_MEMORY_LIMIT=1g` của stack đầy đủ không dùng cho cấu hình mới; thay bằng limit từng container và ngân sách tổng ở mục 2.4.

### 1.4. Kiến trúc mục tiêu của bài thực hành

```mermaid
flowchart TD
    Client[Web / điện thoại / Postman] -->|HTTPS 443| Nginx[Nginx trên VPS]
    Casso[Casso webhook] -->|HTTPS POST| Nginx
    Nginx -->|127.0.0.1:3000| Gateway[Gateway]
    Gateway --> Auth[Auth]
    Gateway --> User[User]
    Gateway --> Chat[Chat HTTP và Socket.IO]
    Gateway --> Todo[Todo]
    Gateway --> Work[Workschedule]
    Gateway --> Canteen[Canteen]
    Gateway --> Payment[Payment]
    Auth --> Redis[Redis]
    Canteen --> Redis
    Auth --> MQ[RabbitMQ]
    MQ --> Mail[Mail]
    MQ --> User
    Payment --> MQ
    MQ --> Canteen
    Payment --> PG[PostgreSQL]
    Auth --> Mongo[MongoDB Atlas: nrapp]
    User --> Mongo
    Chat --> Mongo
    Todo --> Mongo
    Work --> Mongo
    Canteen --> Mongo
    Mail --> SMTP[SMTP ngoài VPS]
    Chat --> Cloudinary[Cloudinary]
    Admin[Máy cá nhân] -->|SSH tunnel| Monitor[Grafana / quản trị nội bộ]
```

Trong phương án này app ghi log JSON ra stdout; không gửi trace/metric OTLP. Prometheus scrape chính nó và Node Exporter; Grafana chỉ xem số liệu host CPU/RAM/disk. Không có lịch sử metrics từng container, dashboard nghiệp vụ hoặc trace phân tán. Sơ đồ lược bớt API User nội bộ; bảng service là danh sách phụ thuộc bổ sung.

## 2. Chọn VPS và dự trù chi phí

### 2.1. Gói nào phù hợp?

| Mục tiêu | Cấu hình dự kiến | Điều kiện |
|---|---|---|
| **Phương án chính tiết kiệm** | **2 vCPU, 4 GB, 50–60 GB SSD, swap 2 GB** | Đủ 9 app + 3 hạ tầng + 3 giám sát; Atlas ngoài VPS; build ngoài VPS; tải thấp, phải đo |
| Khi ngân sách RAM đo thực tế không đủ | Giữ CPU hoặc nâng theo gói, RAM 6–8 GB | Chỉ nâng sau khi đã tắt stack thừa, đo RSS/heap/queue; không tự cắt service nghiệp vụ |
| Có người dùng thật | Quyết định sau khi đo | Đo RAM, p95, queue, dung lượng, backup và mức gián đoạn chấp nhận được |

Chọn region gần người dùng Việt Nam, ưu tiên thử Singapore hoặc Việt Nam. Đặt Atlas gần VPS nếu tier cho phép. Đây là lựa chọn cần kiểm tra độ trễ thực tế; tên region không đảm bảo chất lượng tuyến mạng.

### 2.2. Tham khảo nhà cung cấp

Giá web đối chiếu ngày 11/09/2026, cần xem lại ở trang đặt hàng trước khi trả tiền:

| Lựa chọn | Cách dùng để so sánh |
|---|---|
| DigitalOcean Basic Regular | Gói tham chiếu **2 vCPU/4 GiB/80 GiB SSD: 24 USD/tháng**. Không phải giá thấp nhất toàn thị trường; dùng làm mốc so sánh với VPS Việt Nam. [Bảng giá chính thức](https://www.digitalocean.com/pricing/droplets) |
| Hetzner Cloud | Có lựa chọn Singapore/Europe/USA; kiểm tra cấu hình, region, phí IPv4 và giá tại checkout. Giá mới đã có thay đổi trong 2026, không dựa vào bài viết giá cũ. [Thông tin server](https://docs.hetzner.com/cloud/servers/overview/), [thay đổi giá](https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/) |
| VPS Việt Nam | So sánh gói đáp ứng cấu hình ở trên, KVM, IPv4 riêng, console cứu hộ, khả năng mở SMTP outbound, snapshot và giá gia hạn. Tài liệu không có báo giá đã xác minh cho một hãng cụ thể. |

Khi thanh toán, ghi đủ: VPS + IPv4 nếu tính riêng + thuế + backup/snapshot + domain + Atlas nếu trả phí + phí SMTP/Cloudinary/Casso nếu dùng + lưu backup ngoài VPS + băng thông vượt mức. Không cộng giá khuyến mãi tháng đầu thành chi phí vận hành dài hạn.

### 2.3. Checklist tại trang mua

- [ ] Ubuntu **24.04 LTS x86_64/amd64**, bản server sạch.
- [ ] VPS có quyền root/sudo và chạy Docker được; ưu tiên KVM, tránh shared hosting.
- [ ] Có IPv4 public, outbound Internet và console/recovery khi mất SSH.
- [ ] Có khoảng 50–60 GB đĩa; nếu chỉ 40 GB phải kiểm kê image/backup và phần trống trước khi chọn. Không giữ build cache lớn trên VPS.
- [ ] Trang order cho chọn **Ubuntu**, không chọn Windows Server. Bảng vCPU/RAM/SSD tự nó không cho biết OS; xác nhận Linux, root/sudo và Docker với nhà cung cấp.
- [ ] Có thể nâng RAM/CPU; đã đọc điều kiện resize và hạn chế giảm disk.
- [ ] Có cloud firewall; xác nhận cách tính phí snapshot và lưu backup.
- [ ] Kiểm tra chính sách outbound SMTP **465/587**, vì Mail bắt buộc cho OTP.
- [ ] Thuê theo tháng/giờ cho đợt đầu, đặt nhắc lịch kiểm tra hóa đơn.
- [ ] Hiểu rằng tắt máy có thể vẫn bị tính phí; đọc chính sách xóa tài nguyên khi kết thúc lab.

### 2.4. Ngân sách RAM và disk để kiểm chứng

Đây là **trần cấu hình thử nghiệm**, không phải RAM đã đo hoặc RAM được đặt trước cho mỗi service. Trần thấp có thể khiến app fail ngay lúc boot; cần ghi RSS/heap và nới đúng nơi có bằng chứng, đồng thời giữ ngân sách host.

| Nhóm/service | `mem_limit` dự kiến (MiB) |
|---|---:|
| Gateway / Auth / Chat / Payment / Workschedule | 224 mỗi app = 1120 |
| User / Todo | 192 mỗi app = 384 |
| Mail | 160 |
| Canteen | 256 |
| **Tổng 9 app** | **1920** |
| Redis / RabbitMQ / PostgreSQL | 160 / 384 / 320 = **864** |
| Prometheus / Grafana / Node Exporter | 256 / 192 / 48 = **496** |
| **Tổng trần 15 container** | **3280 MiB** |

VPS quảng cáo 4 GB có thể cung cấp gần 3815 MiB hoặc 4096 MiB; xem `free -m`. Phần còn lại khoảng **535–816 MiB** dành cho OS, Docker, Nginx và quản trị, chưa tính build/import/backup làm tăng tải. Tránh đặt toàn bộ giới hạn vượt RAM rồi kỳ vọng swap cứu mọi trường hợp. Không chạy host Node/PM2 thêm một bản app.

Giới hạn heap Node thấp hơn limit container vì RSS còn có native memory, buffers, code và tiến trình healthcheck. Runtime chạy Node trực tiếp để tránh thêm npm process; Redis đặt `maxmemory` dưới limit và `noeviction` để không âm thầm đẩy phiên/OTP ra khỏi bộ nhớ. RabbitMQ có watermark thấp hơn hard limit; PostgreSQL giảm buffers/connections. Cấu hình cụ thể ở bước 9.

Disk 50–60 GB: dành chỗ OS/Docker, 9 runtime image, DB và **ít nhất 10 GB trống** cho cập nhật/backup tạm. Prometheus giữ 2 ngày và retention size 512 MB cho blocks; WAL/head/compaction vẫn dùng thêm disk, không phải quota toàn volume. Log Docker giới hạn 5 MB × 2 file mỗi container. Sau deploy ổn chỉ giữ một release trước; chuyển backup ra máy cá nhân. [Prometheus: lưu trữ và retention](https://prometheus.io/docs/prometheus/latest/storage/).

Để giảm tiền thuê: dùng Atlas Free cho lab nhỏ theo giới hạn ở bước 7; tận dụng SMTP/Cloudinary test trong quota bạn có; chưa thuê thêm VPS giám sát, managed PostgreSQL hay registry trả phí. Chi phí các dịch vụ ngoài phải xem quota thực tế, không giả định toàn bộ miễn phí vĩnh viễn.

## 3. Chuẩn bị trước khi mua

### 3.1. Điền thông tin của bạn

| Tên sử dụng trong hướng dẫn | Ví dụ cần thay |
|---|---|
| IPv4 VPS | `203.0.113.10` — chỉ là IP tài liệu, không phải VPS thật |
| API domain | `api.example.com` — thay mọi chỗ xuất hiện |
| Tài khoản SSH | `deploy` |
| Thư mục dự án VPS | `/opt/nrapp` |
| Thư mục backend VPS | `/opt/nrapp/backend` |
| Compose project | `nrapp-backend` cho app, `nrapp-monitoring-minimal` cho 3 giám sát; network chung giữ `nrapp-observability` để tương thích Compose backend |
| Database MongoDB | `nrapp` trên **cluster dành riêng cho lab** |
| Database Payment | `nrapp_payment` |

Đoạn lệnh ghi **máy cá nhân** chạy trong terminal máy bạn; đoạn ghi **VPS** chạy sau khi SSH. Shell giả định là Bash; trên Windows dùng WSL hoặc Git Bash cho lệnh Linux. Không dán toàn bộ tài liệu vào terminal một lần.

### 3.2. Chuẩn bị tài khoản và mã nguồn

1. Có tài khoản nhà cung cấp VPS và quản lý DNS của domain; bật MFA cho tài khoản quản trị.
2. Chuẩn bị Atlas lab, SMTP dùng để nhận OTP, Cloudinary test. Có thể hoàn thành cấu hình IP sau khi VPS được tạo.
3. Nếu chưa có domain, vẫn học build và API qua SSH tunnel; cần domain trước bước HTTPS và callback Casso theo lộ trình này.
4. Đọc các file `.env.example`; tạo secret mới cho VPS, không chép nguyên `.env` máy dev.
5. Đảm bảo source có `package-lock.json` của **9 service và `logger/packages/observability`**. Dockerfile dùng `npm ci`; thiếu lockfile là build thất bại.
6. Đưa cả thư mục `backend`, bao gồm `logger/packages/observability`, `logger/observability`, `scripts`, `docker`. Không chỉ upload `src` hoặc `gateway`.

Bản workspace dùng để lập tài liệu không có `.git` tại root/backend. Vì vậy phần upload có cả cách dùng Git và cách `rsync`; không giả định bạn đã có remote repository.

### 3.3. Các điểm phải giải quyết theo kế hoạch

| Phát hiện từ source | Việc phải làm | Bước |
|---|---|---|
| App dùng `restart: "no"` | Override `unless-stopped` cho đủ 9 app | 9 |
| Gateway mặc định `0.0.0.0:3000` | Đổi bind thành `127.0.0.1`; Nginx nhận public traffic | 8, 11 |
| Mail/User có alias `RABBITMQ_*` trong env mẫu; các service đọc alias với ưu tiên khác nhau | Override thống nhất biến RabbitMQ cho Auth/User/Mail/Canteen/Payment | 9 |
| Rate limiter lấy `request.ip`, Gateway chưa cấu hình trust proxy | Thêm cấu hình một proxy tin cậy, Nginx ghi đè forwarded headers | 9, 11 |
| MongoDB một số service cố định `nrapp` | Tách cluster lab, thống nhất database | 7 |
| Auth/Gateway health chỉ kiểm tra liveness | Thử đăng nhập, OTP, profile; không lấy health làm kiểm tra nghiệp vụ | 12 |
| Compose logger kéo nhiều dependency; app vẫn bật OTel | Compose giám sát độc lập, datasource/targets riêng, tắt SDK trên 9 app | 9, 13 |
| Google login chưa thấy kiểm tra `aud` theo client ID trong hàm hiện tại | Bài lab dùng email/OTP; chặn route Google ở Nginx mẫu đến khi hoàn thiện xác minh token | 11 |

## 4. Tạo VPS, SSH và firewall

### 4.1. Tạo SSH key trên máy cá nhân

Nếu đã có key riêng để quản trị server, có thể dùng key đó. Nếu chưa có, chạy trên **máy cá nhân**:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/nrapp_vps -C "nrapp-vps"
cat ~/.ssh/nrapp_vps.pub
```

Đặt passphrase. Chỉ đưa nội dung `.pub` vào phần SSH keys của nhà cung cấp. File không có `.pub` là private key, giữ ở máy cá nhân.

Tạo server theo cấu hình đã chọn, gắn key, đặt hostname `nrapp-lab`. Ghi lại IP và mở thử console của nhà cung cấp.

### 4.2. Đăng nhập lần đầu

**Máy cá nhân**; nếu nhà cung cấp dùng user `ubuntu`, thay `root` và dùng `sudo` ở các lệnh quản trị:

```bash
ssh -i ~/.ssh/nrapp_vps root@203.0.113.10
```

Đối chiếu SSH fingerprint bằng console nếu được nhà cung cấp cung cấp. **VPS**, đang là root:

```bash
apt update
apt upgrade -y
apt install -y sudo curl ca-certificates git rsync ufw nginx jq unzip htop dnsutils netcat-openbsd python3 openssl nano less ripgrep
hostnamectl set-hostname nrapp-lab
timedatectl set-timezone Asia/Ho_Chi_Minh
timedatectl set-ntp true
timedatectl status
adduser deploy
usermod -aG sudo deploy
install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
install -m 600 -o deploy -g deploy /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
```

Nếu đăng nhập ban đầu là `ubuntu`, copy file `authorized_keys` từ home của user đó bằng `sudo`, không dùng đường dẫn `/root` khi không có file.

**Giữ phiên root đang mở**, mở terminal thứ hai trên máy cá nhân:

```bash
ssh -i ~/.ssh/nrapp_vps deploy@203.0.113.10
sudo -v
```

Chỉ sau khi key của `deploy` và `sudo` hoạt động mới tắt đăng nhập root/password.

### 4.3. Giới hạn SSH

**VPS, user deploy**:

```bash
sudo tee /etc/ssh/sshd_config.d/00-nrapp.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
EOF
sudo sshd -t
sudo sshd -T | rg 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication'
```

Nếu chưa có `rg` trên VPS, cài `sudo apt install -y ripgrep` rồi chạy lệnh kiểm tra trên. Nếu cấu hình hiệu lực không đúng, kiểm tra các file SSH khác và `Match` block trước khi reload.

```bash
sudo systemctl reload ssh
```

Thử **một kết nối deploy mới** lần nữa. Không đóng phiên dự phòng trước khi thử xong. Nếu máy yêu cầu reboot sau nâng cấp hệ điều hành, thực hiện lúc này khi chưa có dữ liệu ứng dụng.

### 4.4. Hai lớp firewall

Trong cloud firewall của nhà cung cấp, cho phép inbound:

| Port | Nguồn | Mục đích |
|---|---|---|
| TCP 22 | IP máy quản trị `/32` nếu ổn định | SSH; cập nhật rule khi đổi mạng |
| TCP 80 | Internet | HTTP redirect và ACME cấp/gia hạn certificate |
| TCP 443 | Internet | API HTTPS và WebSocket |

Để outbound hoạt động cho DNS, NTP, HTTPS, Atlas, SMTP; không tự áp dụng outbound deny-all trong bài đầu. Cấu hình IPv6 tương ứng nếu bật IPv6.

**VPS**:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status verbose
```

Cloud firewall thu hẹp nguồn SSH; UFW ở đây giữ OpenSSH để không tự khóa khi IP máy cá nhân thay đổi. Nếu chỉ dùng UFW, thay rule SSH bằng IP quản trị và có console cứu hộ.

Docker có thể đưa published port đi vòng qua UFW. Do đó phải bind cổng nội bộ ở `127.0.0.1` và kiểm tra từ máy bên ngoài, không chỉ nhìn `ufw status`. [Docker: firewall và published ports](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

**Hoàn thành bước 4 khi:** deploy SSH bằng key được, sudo được, thời gian đồng bộ, có console dự phòng, chưa mở public các port database/app nội bộ.

## 5. Cài Docker, Node và công cụ

### 5.1. Docker Engine từ repository chính thức

**VPS, deploy**. Các lệnh cho Ubuntu **24.04 Noble amd64** đã chọn, không dùng nguyên cho distro khác:

```bash
sudo apt install -y ca-certificates curl ripgrep
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<'EOF'
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker deploy
```

Đăng xuất rồi SSH lại để nhận group `docker`; sau đó:

```bash
docker version
docker compose version
docker run --rm hello-world
systemctl is-enabled docker
```

Quyền group Docker cho phép điều khiển host gần tương đương root; chỉ cấp cho tài khoản quản trị. Dùng plugin `docker compose` hiện hành. [Hướng dẫn Docker Ubuntu](https://docs.docker.com/engine/install/ubuntu/).

### 5.2. Node 22 trên host để chạy script quản trị

App vẫn chạy trong container Node 22. **Node host là tùy chọn**, chỉ cài nếu muốn chạy payment smoke script ở VPS; helper `dc`/`mc` và Docker deploy không cần Node host. Build/test chính ở máy cá nhân. Không dùng script điều phối full logger trên VPS tiết kiệm.

**VPS, deploy, không sudo**:

```bash
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh -o /tmp/nrapp-install-nvm.sh
less /tmp/nrapp-install-nvm.sh
bash /tmp/nrapp-install-nvm.sh
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"
nvm install 22
nvm alias default 22
node --version
npm --version
```

Trong `less`, bấm `q` để thoát sau khi xem. NVM tải bản vá Node 22 tại lúc cài; ghi lại `node --version`. Không dùng `sudo npm`. [NVM: cài đặt và sử dụng](https://github.com/nvm-sh/nvm#installing-and-updating).

### 5.3. Đo tài nguyên ban đầu

```bash
free -h
df -h /
lsblk
nproc
swapon --show
```

Với VPS 4 GB chưa có swap, tạo **2 GB swap** làm khoảng đệm lúc quản trị/tăng tải ngắn. Chỉ chạy khối sau nếu `/swapfile` chưa tồn tại và đủ disk; không coi swap là 2 GB RAM bổ sung để chạy thường trực:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
printf '%s\n' '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
swapon --show
```

Swap không thay RAM. Nếu máy liên tục swap trong lúc chạy bình thường, giảm tải hoặc nâng RAM.

## 6. Đưa source lên VPS

### 6.1. Tạo thư mục

**VPS**:

```bash
sudo mkdir -p /opt/nrapp /opt/nrapp-backups
sudo chown deploy:deploy /opt/nrapp /opt/nrapp-backups
chmod 700 /opt/nrapp-backups
```

Chọn **một** trong hai cách dưới đây, đích cuối cùng phải là `/opt/nrapp/backend/compose.yaml`.

### 6.2. Cách A — Git nếu đã có repository

Clone bằng quyền đọc repository. Với private repo, dùng deploy key chỉ đọc hoặc luồng xác thực của Git provider; không nhúng token vào URL lệnh.

```bash
# VPS: thay URL bằng repo chứa thư mục backend.
git clone REPLACE_WITH_REPOSITORY_URL /opt/nrapp
cd /opt/nrapp
git rev-parse HEAD
test -f backend/compose.yaml
```

Nếu repository chỉ chứa nội dung backend, clone vào `/opt/nrapp/backend`. Kiểm tra credential không bị track trước khi push ở máy dev; `.gitignore` không tự bỏ file đã track từ trước.

### 6.3. Cách B — rsync từ workspace hiện tại

**Máy cá nhân**, chạy tại thư mục chứa `backend/`:

```bash
rsync -avz \
  -e 'ssh -i ~/.ssh/nrapp_vps' \
  --exclude='node_modules/' --exclude='dist/' --exclude='coverage/' \
  --exclude='.git/' --exclude='.env' --exclude='.env.local' \
  --exclude='*.log' --exclude='logs/' \
  --exclude='logger/observability/alertmanager/secrets/*' \
  --include='*/' --include='.env.example' --exclude='.env.*' \
  backend/ deploy@203.0.113.10:/opt/nrapp/backend/
```

Lệnh giữ `.env.example`, bỏ env thật và secret alert đã cấu hình local. Không thêm `--delete` vào lần đầu. Nếu dùng cấu hình cảnh báo riêng sau này, tạo secret trực tiếp trên VPS.

### 6.4. Xác nhận cấu trúc

**VPS**:

```bash
cd /opt/nrapp/backend
test -f compose.yaml
test -f docker/node-service.Dockerfile
test -f logger/packages/observability/package-lock.json
test -f logger/observability/prometheus/prometheus.yaml
for service in gateway auth user mail chat todo workschedule canteen payment; do
  test -f "$service/package-lock.json" || printf 'THIEU lockfile: %s\n' "$service"
done
```

Không chạy `npm install` ở backend root với kỳ vọng cài đủ 9 app: root hiện chỉ có script điều phối, không khai báo npm workspaces cho toàn bộ service.

## 7. Chuẩn bị MongoDB và dịch vụ ngoài

### 7.1. MongoDB Atlas lab

1. Tạo project/cluster riêng cho bài thực hành. Không trỏ lab vào cluster đang chứa dữ liệu thật.
2. Chọn region gần VPS. Free tier có thể dùng cho dữ liệu thử nhỏ; kiểm tra tier có ở region bạn chọn.
3. Trong Database Access, tạo user ứng dụng với quyền `readWrite` trên database **`nrapp`**. Dùng password mạnh; chuỗi hex thuận tiện khi tạo URI.
4. Trong Network Access, thêm **IP public outbound thực tế của VPS `/32`**. Nếu nhà cung cấp có NAT/Reserved IP, outbound có thể khác IP nhận SSH; đối chiếu tài liệu của họ. Không mở `0.0.0.0/0` làm cấu hình mặc định.
5. Chỉ thêm IP máy cá nhân khi cần Compass/mongosh và xóa quyền truy cập không còn dùng.
6. Lấy connection string từ Connect → Drivers; đặt database `/nrapp` trước phần `?`.
7. Dùng URI đó cho Auth/User/Chat/Todo/Workschedule/Canteen. Không dùng `localhost` hoặc `127.0.0.1` trong URI Atlas.

Ví dụ cấu trúc, **không dùng nguyên placeholder**:

```dotenv
MONGO_URL=mongodb+srv://nrapp_lab:REPLACE_PASSWORD@REPLACE_CLUSTER.mongodb.net/nrapp?retryWrites=true&w=majority&maxPoolSize=10&minPoolSize=0
MONGO_DB_NAME=nrapp
```

`maxPoolSize=10&minPoolSize=0` là giới hạn khởi đầu cho mỗi client, giúp giảm kết nối và bộ nhớ phía sáu service trong lab. Nó không giới hạn tổng connection toàn cluster ở 10; vẫn có monitoring connections và công cụ quản trị. Giữ cùng cấu hình URI trên sáu app; kiểm tra độ trễ chờ pool khi tăng tải.

Atlas Free có giới hạn tài nguyên/kết nối và không cung cấp backup tự động như các tier có backup; tài liệu chính thức hiện nêu tối đa 500 connections. Cần tự dump/restore trong bài lab. [Atlas Free limits](https://www.mongodb.com/docs/atlas/reference/free-shared-limitations/).

Nếu password có ký tự đặc biệt, URL-encode phần username/password khi đưa vào URI. Giữ nguyên URI trong dấu nháy đơn ở dotenv nếu cần tránh nội suy `$`; không commit hoặc gửi URI chứa credential vào log/chat.

### 7.2. SMTP — làm trước khi chờ OTP

1. Chọn SMTP account dành cho thử nghiệm. Nếu Gmail, dùng App Password khi tài khoản đủ điều kiện; không dùng mật khẩu đăng nhập thường làm `SMTP_PASS`.
2. Điền `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM` trong `mail/.env`.
3. Port 465 thường dùng TLS ngay đầu kết nối: `SMTP_SECURE=true`. Với nhà cung cấp hướng dẫn STARTTLS port 587: `SMTP_SECURE=false`.
4. Kiểm tra kết nối outbound từ **VPS** đến host thực tế:

```bash
nc -vz smtp.gmail.com 465
```

5. Port mở mới xác nhận kết nối TCP; readiness Mail và email nhận thật ở bước 12 mới xác nhận credential và gửi thư.
6. Nếu bị nhà cung cấp VPS chặn outbound SMTP, yêu cầu họ mở hoặc chọn SMTP/nhà cung cấp đáp ứng. Không mở inbound 465 trên VPS để chữa lỗi outbound.

### 7.3. Cloudinary

1. Chuẩn bị tài khoản/môi trường Cloudinary thử nghiệm.
2. Điền đủ `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` trong `chat/.env`.
3. Giữ API secret ở backend; frontend chỉ gọi API upload hiện có.
4. Upload một ảnh nhỏ khi test Chat và kiểm tra cả record message lẫn tài nguyên Cloudinary.

### 7.4. Payment / Casso

1. Trước hết dựng database và endpoint với secret lab; chưa cấu hình ngân hàng thật vẫn kiểm tra được health và xác thực request.
2. Để dùng VietQR, điền đúng bank ID, số tài khoản, chủ tài khoản và template theo cấu hình `payment/.env.example`.
3. Khi đã có HTTPS, cấu hình Casso V2 callback tới **`https://api.example.com/api/payment/webhooks/casso`**.
4. `CASSO_WEBHOOK_SECRET` phải là secret đúng phía Casso, độc lập `PAYMENT_INTERNAL_SECRET` dùng giữa Gateway và Payment.
5. Source có alias `/api/payment/webhook/casso` và `/api/payment/callback`; chọn đường dẫn plural `webhooks/casso` nhất quán cho tài liệu/test.
6. Không đặt Cloudflare Access/login challenge trước callback máy gọi máy. Giữ xác thực chữ ký ở Payment.

Chi tiết hợp đồng dự án: [Payment Service Guide](PAYMENT_SERVICE_GUIDE.md), [hướng dẫn Casso/Cloudflare hiện có](HUONG_DAN_SETUP_BACKEND_THANH_TOAN_CASSO_CLOUDFLARE.md). Khi UI/tính năng phía Casso thay đổi, kiểm tra lại tài liệu nhà cung cấp tại lúc cấu hình.

## 8. Tạo env và đồng bộ secret

### 8.1. Tạo từ file mẫu, không ghi đè env đã có

**VPS**:

```bash
cd /opt/nrapp/backend
umask 077
test -f .env || cp .env.example .env
test -f logger/.env || cp logger/.env.example logger/.env
for service in gateway auth user mail chat todo workschedule canteen payment; do
  test -f "$service/.env" || cp "$service/.env.example" "$service/.env"
done
chmod 600 .env logger/.env gateway/.env auth/.env user/.env mail/.env chat/.env todo/.env workschedule/.env canteen/.env payment/.env
```

Dùng `nano .env` hoặc editor qua SSH. Sinh mỗi secret bằng `openssl rand -hex 32`; mỗi lần chạy cho một giá trị mới. Lưu trong password manager, chỉ dán cùng giá trị ở các đầu cần chia sẻ.

### 8.2. Sửa `backend/.env`

Giữ các port mặc định của file mẫu; cập nhật các mục sau:

```dotenv
COMPOSE_PROJECT_NAME=nrapp-backend
NRAPP_IMAGE_TAG=vps-lab-001
OBSERVABILITY_COMPOSE_PROJECT_NAME=nrapp-observability
OBSERVABILITY_NETWORK_NAME=nrapp-observability
GATEWAY_BIND_IP=127.0.0.1
GATEWAY_HOST_PORT=3000

RABBITMQ_USER=nrapp_lab
RABBITMQ_PASSWORD=REPLACE_WITH_RANDOM_HEX
PAYMENT_POSTGRES_USER=nrapp_payment
PAYMENT_POSTGRES_PASSWORD=REPLACE_WITH_ANOTHER_RANDOM_HEX
PAYMENT_POSTGRES_DB=nrapp_payment

USER_INTERNAL_SECRET=REPLACE_WITH_RANDOM_HEX
CHAT_INTERNAL_SECRET=REPLACE_WITH_RANDOM_HEX
TODO_INTERNAL_SECRET=REPLACE_WITH_RANDOM_HEX
WORKSCHEDULE_INTERNAL_SECRET=REPLACE_WITH_RANDOM_HEX
CANTEEN_INTERNAL_SECRET=REPLACE_WITH_RANDOM_HEX
PAYMENT_INTERNAL_SECRET=REPLACE_WITH_RANDOM_HEX

OTEL_TRACES_SAMPLER_ARG=0
OTEL_METRIC_EXPORT_INTERVAL=15000
OTEL_METRIC_EXPORT_TIMEOUT=10000
```

**Mỗi `REPLACE_...` phải thay bằng giá trị thật khác nhau**, ngoại trừ những nơi cố ý chia sẻ cùng secret. Sampler 0 ở đây chưa đủ tắt metrics/exporter; bắt buộc inject `OTEL_SDK_DISABLED=true` bằng override bước 9. Các biến root của full observability có thể còn trong mẫu nhưng không khởi chạy stack nếu chỉ dùng `dc`/`mc`.

`AUTH_INTERNAL_SECRET` và `JWT_SECRET` chưa được Compose gốc lấy từ root env để inject. Chỉ thêm chúng vào root env sẽ **không đủ**; điền ở env service như bảng dưới.

### 8.3. Bảng ghép secret — kiểm tra trước khi build

| Secret | Các nơi phải cùng giá trị | Nguồn runtime khi dùng Compose trong tài liệu |
|---|---|---|
| `JWT_SECRET` | `gateway/.env`, `auth/.env`, `chat/.env`; điền nhất quán cả `user/.env` | `.env` của từng service |
| `AUTH_INTERNAL_SECRET` | `gateway/.env` và `auth/.env` | `.env` của từng service; chủ động dùng secret riêng thay fallback JWT |
| `USER_INTERNAL_SECRET` | Gateway, User, Todo, Workschedule | Compose inject từ `backend/.env` |
| `CHAT_INTERNAL_SECRET` | Gateway, Chat | Compose inject từ root |
| `TODO_INTERNAL_SECRET` | Gateway, Todo | Compose inject từ root |
| `WORKSCHEDULE_INTERNAL_SECRET` | Gateway, Workschedule | Compose inject từ root |
| `CANTEEN_INTERNAL_SECRET` | Gateway, Canteen | Compose inject từ root |
| `PAYMENT_INTERNAL_SECRET` | Gateway, Payment | Compose inject từ root |
| `CASSO_WEBHOOK_SECRET` | Casso và Payment | `payment/.env` |
| RabbitMQ user/password | Broker, Auth, User, Mail, Canteen, Payment | Root env + override chung cho 5 app ở bước 9 |
| PostgreSQL password | PostgreSQL và Payment | Root env; không chạy PostgreSQL exporter trên VPS này |

Đồng bộ bản sao internal secret trong service env với root để sau này chạy script trực tiếp không dùng nhầm giá trị cũ. Khi chạy Compose, `environment:` sẽ ghi đè `env_file:`. `backend/.env` chủ yếu phục vụ **nội suy Compose**, không tự phát tán tất cả biến cho mọi container.

### 8.4. Bảng việc cần sửa theo service

| File | Giá trị bạn cần điền |
|---|---|
| `auth/.env` | Atlas URI, `MONGO_DB_NAME=nrapp`, JWT và Auth internal secret |
| `user/.env` | Atlas URI, `MONGO_DB_NAME=nrapp`, JWT/internal secret nhất quán |
| `chat/.env` | Atlas URI, JWT, Cloudinary; `LOG_LEVEL=info` |
| `todo/.env` | Atlas URI, internal secrets; `LOG_LEVEL=info` |
| `workschedule/.env` | Atlas URI, internal secrets; `LOG_LEVEL=info` |
| `canteen/.env` | Atlas URI **có `/nrapp`**, internal secret |
| `gateway/.env` | JWT, Auth internal secret; giữ rate limit khởi đầu 120 request/60 giây rồi đo lại |
| `mail/.env` | SMTP thật cho lab; queue giữ tên `send-otp`, retry, DLQ như mẫu |
| `payment/.env` | Casso secret, cấu hình tài khoản nhận QR, internal secret |
| `logger/.env` | Mật khẩu Grafana mới, network khớp root; retention minimal khai báo trực tiếp trong Compose mới |

Trong các service env, đổi `NODE_ENV=production` nếu có và `LOG_LEVEL=info`. Compose gốc inject cấu hình production và OTLP container URL; override VPS thêm `OTEL_SDK_DISABLED=true` nên URL Collector đó không được export tới. Các dòng localhost upstream/Redis/Postgres được Compose override; riêng **MongoDB, SMTP, Cloudinary, JWT/Auth secret vẫn phải điền thật**.

Logger lab:

```dotenv
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=REPLACE_WITH_RANDOM_PASSWORD
OBSERVABILITY_NETWORK_NAME=nrapp-observability
```

Các biến full logger còn trong file mẫu không được minimal Compose dùng. Không khởi động Loki/Alloy/Jaeger/Collector/Alertmanager. Giữ file gốc để sau này học ở local.

### 8.5. Kiểm tra env mà không in secret

```bash
cd /opt/nrapp/backend
python3 - <<'PY'
from pathlib import Path
files = [Path('.env'), Path('logger/.env')]
files += [Path(s) / '.env' for s in
          ['gateway', 'auth', 'user', 'mail', 'chat', 'todo', 'workschedule', 'canteen', 'payment']]
markers = ('REPLACE_', 'CHANGE_ME', 'replace_with', 'replace-with', 'your_cloudinary', 'your-email')
for path in files:
    for number, line in enumerate(path.read_text().splitlines(), 1):
        if line.lstrip().startswith('#') or '=' not in line:
            continue
        key, value = line.split('=', 1)
        if any(marker in value for marker in markers):
            print(f'{path}:{number}: còn placeholder tại {key.strip()}')
PY
```

Đây là bộ lọc hỗ trợ, không kiểm tra credential hợp lệ. Các biến tùy chọn có thể để trống theo file mẫu, ví dụ `CASSO_WEBHOOK_PREVIOUS_SECRET`; các mục chưa dùng như pgAdmin vẫn cần hoàn thiện trước khi bật profile đó.

## 9. Cấu hình riêng cho VPS

### 9.1. Tạo `backend/compose.vps.yaml`

Tài liệu chọn override riêng để giữ nguyên Compose dev. **VPS**, tại `/opt/nrapp/backend`, tạo file:

```bash
nano compose.vps.yaml
```

Nội dung đầy đủ:

```yaml
x-vps-app: &vps-app
  restart: unless-stopped
  stop_grace_period: 30s
  command: ["node", "--require", "@nrapp/observability/register", "dist/main.js"]
  mem_limit: 224m
  healthcheck:
    interval: 30s
    start_period: 40s
  logging: &vps-logging
    driver: local
    options:
      max-size: 5m
      max-file: "2"

x-vps-env: &vps-env
  OTEL_SDK_DISABLED: "true"
  OTEL_METRICS_EXPORTER: none
  LOG_FORMAT: json
  LOG_LEVEL: info
  NODE_OPTIONS: --max-old-space-size=128

x-vps-rabbitmq: &vps-rabbitmq
  RABBITMQ_HOST: rabbitmq
  RABBITMQ_PORT: "5672"
  RABBITMQ_AMQP_HOST_PORT: "5672"
  RABBITMQ_USER: ${RABBITMQ_USER:?Set RABBITMQ_USER}
  RABBITMQ_PASSWORD: ${RABBITMQ_PASSWORD:?Set RABBITMQ_PASSWORD}

services:
  gateway:
    <<: *vps-app
    image: nrapp/gateway:${NRAPP_IMAGE_TAG:-vps-lab-001}
    environment: *vps-env
  auth:
    <<: *vps-app
    image: nrapp/auth:${NRAPP_IMAGE_TAG:-vps-lab-001}
    environment:
      <<: [*vps-env, *vps-rabbitmq]
  user:
    <<: *vps-app
    image: nrapp/user:${NRAPP_IMAGE_TAG:-vps-lab-001}
    mem_limit: 192m
    environment:
      <<: [*vps-env, *vps-rabbitmq]
  mail:
    <<: *vps-app
    image: nrapp/mail:${NRAPP_IMAGE_TAG:-vps-lab-001}
    mem_limit: 160m
    environment:
      <<: [*vps-env, *vps-rabbitmq]
      NODE_OPTIONS: --max-old-space-size=96
      MAIL_PREFETCH: "2"
  chat:
    <<: *vps-app
    image: nrapp/chat:${NRAPP_IMAGE_TAG:-vps-lab-001}
    environment: *vps-env
  todo:
    <<: *vps-app
    image: nrapp/todo:${NRAPP_IMAGE_TAG:-vps-lab-001}
    mem_limit: 192m
    environment: *vps-env
  workschedule:
    <<: *vps-app
    image: nrapp/workschedule:${NRAPP_IMAGE_TAG:-vps-lab-001}
    environment: *vps-env
  canteen:
    <<: *vps-app
    image: nrapp/canteen:${NRAPP_IMAGE_TAG:-vps-lab-001}
    mem_limit: 256m
    environment:
      <<: [*vps-env, *vps-rabbitmq]
      NODE_OPTIONS: --max-old-space-size=160
  payment:
    <<: *vps-app
    image: nrapp/payment:${NRAPP_IMAGE_TAG:-vps-lab-001}
    environment:
      <<: [*vps-env, *vps-rabbitmq]

  redis:
    mem_limit: 160m
    logging: *vps-logging
    command: ["redis-server", "--appendonly", "yes", "--save", "60", "1", "--maxmemory", "48mb", "--maxmemory-policy", "noeviction"]
  rabbitmq:
    mem_limit: 384m
    logging: *vps-logging
    volumes:
      - type: bind
        source: ./docker/rabbitmq.vps.conf
        target: /etc/rabbitmq/rabbitmq.conf
        read_only: true
        bind:
          create_host_path: false
  payment-postgres:
    mem_limit: 320m
    logging: *vps-logging
    command:
      - postgres
      - -c
      - shared_buffers=64MB
      - -c
      - work_mem=2MB
      - -c
      - maintenance_work_mem=32MB
      - -c
      - max_connections=30
      - -c
      - max_parallel_workers=0
```

Mail cần override này vì [environment.ts](../backend/mail/src/config/environment.ts) ưu tiên uppercase của `.env.example` trước legacy mixed-case do Compose gốc inject. User cũng có `RABBITMQ_USER=guest`/`RABBITMQ_PASSWORD=guest` trong env mẫu và [RabbitMQ service](../backend/user/src/modules/rabbitmq/rabbitmq.service.ts) ưu tiên các alias đó. Áp override chung cho cả 5 app dùng broker để host/user/password thống nhất. Các biến `Rabbitmq_*` từ Compose gốc vẫn được giữ khi merge.

`RABBITMQ_AMQP_HOST_PORT=5672` trong **environment container** ở đây là để tương thích tên biến cũ trong source: kết nối nội bộ luôn tới `rabbitmq:5672`. Nếu đổi port publish ở root `.env`, không mang host port đó vào kết nối container-to-container.

Tạo `backend/docker/rabbitmq.vps.conf` trước khi `up`:

```ini
vm_memory_high_watermark.absolute = 256MiB
disk_free_limit.absolute = 1GB
```

RabbitMQ watermark là ngưỡng **chặn publisher khi RAM tăng**, không phải giới hạn cứng; `mem_limit:384m` mới là trần container. Nếu có memory alarm, kiểm tra consumer/backlog, không tăng watermark sát hard limit. [RabbitMQ: memory threshold](https://www.rabbitmq.com/docs/memory).

`NODE_OPTIONS` chỉ giới hạn old-space heap, không giới hạn toàn bộ RSS; `mem_limit` áp dụng cả Node và tiến trình healthcheck. Chạy Node trực tiếp vẫn preload observability như Gateway và giữ shutdown signal qua `init:true` trong base Compose. Nếu thay đổi start script về sau, kiểm tra lại command override này. Tắt SDK dựa trên nhánh `OTEL_SDK_DISABLED` có sẵn trong [sdk.js](../backend/logger/packages/observability/sdk.js), không xóa thư viện hoặc import.

Giữ Redis AOF/RDB và `noeviction`: khi đạt `maxmemory`, một số lệnh ghi sẽ bị từ chối; cần xử lý dung lượng, không dùng `allkeys-lru` tùy tiện cho OTP/refresh token. Khoảng trống trên 48 MB dùng cho overhead/fork/rewrite, không đảm bảo dữ liệu lớn luôn vừa. PostgreSQL `work_mem` tính theo từng thao tác/connection, không phải tổng DB; max_connections 30 dành cho một Payment replica và quản trị ít kết nối. Các limit này phải được nghiệm thu ở mục 13.4, không chỉ thấy container khởi động.

### 9.2. Tạo lệnh `dc` để luôn dùng cả hai file

**VPS**, tạo helper không chứa secret:

```bash
mkdir -p /home/deploy/bin
cat > /home/deploy/bin/dc <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd /opt/nrapp/backend
exec docker compose --env-file .env -f compose.yaml -f compose.vps.yaml "$@"
EOF
chmod 755 /home/deploy/bin/dc
export PATH="/home/deploy/bin:$PATH"
```

Thêm một lần dòng sau vào `~/.bashrc` để phiên SSH sau cũng có `dc`:

```bash
export PATH="/home/deploy/bin:$PATH"
```

Từ đây mọi thao tác app dùng **`dc`**. Các lệnh `npm run docker:up`, `docker:start`, `infra:up` hiện tại không biết file `compose.vps.yaml`; không dùng chúng thay lệnh của hướng dẫn. Script `npm run dev` còn tự dừng app container, không chạy trên VPS này.

`unless-stopped` giúp container khởi động lại khi tiến trình thoát hoặc Docker khởi động lại, trừ container đã chủ động stop. **Docker không tự restart chỉ vì healthcheck chuyển `unhealthy`**; cần điều tra và xử lý riêng.

### 9.3. Tạo stack giám sát độc lập đúng 3 container

**VPS**, tạo các thư mục mới; giữ nguyên toàn bộ `logger/compose.yaml` và provisioning cũ:

```bash
cd /opt/nrapp/backend
mkdir -p logger/vps-minimal/grafana/provisioning/datasources
nano logger/compose.vps-minimal.yaml
```

Nội dung file **độc lập**, không phải override của full logger. Các tag image lấy từ repo hiện tại:

```yaml
name: nrapp-monitoring-minimal

x-minimal-service: &minimal-service
  restart: unless-stopped
  networks: [observability]
  security_opt: ["no-new-privileges:true"]
  logging:
    driver: local
    options:
      max-size: 5m
      max-file: "2"

services:
  prometheus:
    <<: *minimal-service
    image: prom/prometheus:v3.12.0
    mem_limit: 256m
    cpus: 0.5
    command:
      - --config.file=/etc/prometheus/prometheus.yaml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=2d
      - --storage.tsdb.retention.size=512MB
      - --query.max-concurrency=2
    ports: ["127.0.0.1:9090:9090"]
    volumes:
      - type: bind
        source: ./vps-minimal/prometheus.yaml
        target: /etc/prometheus/prometheus.yaml
        read_only: true
        bind:
          create_host_path: false
      - prometheus_data:/prometheus
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:9090/-/ready"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s
  grafana:
    <<: *minimal-service
    image: grafana/grafana:13.2.0
    mem_limit: 192m
    cpus: 0.5
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER:-admin}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:?Set GRAFANA_ADMIN_PASSWORD}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_PLUGINS_PREINSTALL_DISABLED: "true"
      GF_ANALYTICS_REPORTING_ENABLED: "false"
    ports: ["127.0.0.1:3001:3000"]
    volumes:
      - grafana_data:/var/lib/grafana
      - type: bind
        source: ./vps-minimal/grafana/provisioning
        target: /etc/grafana/provisioning
        read_only: true
        bind:
          create_host_path: false
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 45s
  node-exporter:
    <<: *minimal-service
    image: prom/node-exporter:v1.11.1
    mem_limit: 48m
    cpus: 0.2
    pid: host
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/host/root
      - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/containers/storage/.+)(/.*)?
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro,rslave
    expose: ["9100"]
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:9100/metrics"]
      interval: 60s
      timeout: 5s
      retries: 3

networks:
  observability:
    external: true
    name: ${OBSERVABILITY_NETWORK_NAME:-nrapp-observability}

volumes:
  prometheus_data:
  grafana_data:
```

Tạo `backend/logger/vps-minimal/prometheus.yaml`; không copy toàn bộ scrape targets/rule files từ cấu hình cũ:

```yaml
global:
  scrape_interval: 60s
  evaluation_interval: 60s
  scrape_timeout: 10s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]
  - job_name: node-exporter
    static_configs:
      - targets: ["node-exporter:9100"]
```

Không scrape `/health` như Prometheus metrics: response JSON của health không phải định dạng metrics. Không có Collector/Blackbox trong cấu hình mới nên bỏ application-metrics và backend-readiness targets cũ.

Tạo `backend/logger/vps-minimal/grafana/provisioning/datasources/prometheus.yaml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prometheus-minimal
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: 60s
```

Chỉ mount provisioning mới này. Dashboard/alerting cũ có thể phụ thuộc Loki, Jaeger, cAdvisor hoặc Blackbox; không mount chúng rồi coi `No data` là lỗi ứng dụng. Tạo dashboard CPU/RAM/disk ở bước 13.

Tạo helper `mc` trên VPS, dùng **đúng một file Compose minimal**:

```bash
cat > /home/deploy/bin/mc <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd /opt/nrapp/backend
exec docker compose --project-name nrapp-monitoring-minimal \
  --env-file .env --env-file logger/.env \
  -f logger/compose.vps-minimal.yaml "$@"
EOF
chmod 755 /home/deploy/bin/mc
```

Các file YAML provisioning/Prometheus phải đọc được bởi container user: `chmod 644` cho chúng, thư mục `755`; riêng `.env` vẫn `600`. Không có secret trong YAML mẫu ngoài env interpolation. Đường dẫn bind resolve tương đối thư mục `logger`, không phải backend root.

### 9.4. Sửa IP client trước khi đưa Gateway qua Nginx

Source [rate-limit.middleware.ts](../backend/gateway/src/common/middleware/rate-limit.middleware.ts) lấy `request.ip`. Nếu chỉ thêm Nginx mà không cấu hình proxy, nhiều client có thể bị tính chung IP Nginx và cùng chạm ngưỡng 120 request/phút.

**Thay đổi code cần thực hiện trong bài triển khai, chưa được áp dụng bởi tài liệu này:** trong `gateway/src/main.ts`, ngay sau `NestFactory.create(...)`, thêm:

```typescript
// VPS topology: Internet -> host Nginx -> Gateway container.
// Gateway host port must remain bound to 127.0.0.1.
app.getHttpAdapter().getInstance().set('trust proxy', 1);
```

Cấu hình này áp dụng cho **đúng một HTTP reverse proxy Nginx**. Docker NAT không phải thêm một HTTP proxy. Nginx ở bước 11 ghi đè `X-Forwarded-For` bằng `$remote_addr`, và chỉ Nginx được làm cửa vào public. Không dùng `trust proxy=true` chung chung; không áp dụng nguyên cấu hình này khi thêm Cloudflare proxy/CDN/Tunnel mà chưa xét lại đường đi IP. [Express: behind proxies](https://expressjs.com/en/guide/behind-proxies/).

Đây là hạng mục proxy riêng từ kế hoạch triển khai ban đầu, **không phải sửa source để giảm RAM**. Tối ưu monitoring/heap/queue ở trên chỉ cần config. Thực hiện thay đổi proxy ở source quản lý của bạn trước khi build image; test trên **máy cá nhân**, tại thư mục dự án:

```bash
cd backend
npm ci --prefix logger/packages/observability --no-audit --no-fund
npm ci --prefix gateway --no-audit --no-fund
npm --prefix gateway test
```

Test hiện có không thay bài kiểm tra proxy thực tế. Sau HTTPS, dùng hai mạng khác nhau (Wi-Fi và 4G) kiểm tra header `x-ratelimit-remaining` có bucket riêng; người cùng NAT vẫn có thể dùng chung bucket theo thiết kế per-IP.

### 9.5. Kiểm tra Compose trước khi khởi động

```bash
cd /opt/nrapp/backend
dc --profile app config --quiet
mc config --quiet
dc --profile app config --services
mc config --services
```

Hai cấu hình phải exit 0. `dc` có 12 service: 9 app + 3 hạ tầng; `mc` phải chỉ có **prometheus, grafana, node-exporter**. Không paste `docker compose config` nguyên bản lên nơi công khai vì output có thể chứa env/secret đã nội suy. Kiểm tra `OTEL_SDK_DISABLED=true` và `mem_limit` đã merge trên đủ 9 app; đặt biến chỉ ở root `.env` chưa đủ inject vào app.

Giữ nguyên project name sau khi có dữ liệu: đổi tên project có thể tạo volume khác, nhìn giống mất database dù volume cũ vẫn còn.

Nếu trước đó **đã bật full logger trên chính VPS**, dừng container cũ trước khi bật minimal để không trùng port và tốn RAM. Chỉ chạy lệnh sau khi xác nhận đúng project cũ `nrapp-observability` (thay tên nếu trước đây đã đổi):

```bash
docker ps -a --filter label=com.docker.compose.project=nrapp-observability
docker compose --project-name nrapp-observability \
  --env-file .env --env-file logger/.env -f logger/compose.yaml stop
```

Dùng `stop` để giữ volume/network; không `down -v`, không xóa volume cũ. Minimal dùng project và volume riêng, dashboard/history cũ không tự chuyển. VPS mới bỏ qua bước chuyển đổi này. Các container full đã stop không tự bật lại theo `unless-stopped`; kiểm tra sau reboot. Network được tạo thủ công ở bước 10, giữ tên để app không cần thay source.

## 10. Build và khởi động theo thứ tự

### 10.1. Build image ở máy cá nhân — lộ trình chính tiết kiệm

Build ở máy có đủ RAM thay vì vừa chạy 15 container vừa biên dịch trên VPS 4 GB. Dùng Docker Desktop/Engine ở máy cá nhân; ở Windows dùng WSL Bash. Source, Dockerfile và thư viện logger vẫn giữ nguyên; thêm sửa proxy ở bước 9.4 nếu chọn topology Nginx đó.

**Máy cá nhân**, tại thư mục chứa `backend/`, build tuần tự đúng kiến trúc VPS:

```bash
cd backend
(
  set -euo pipefail
  NRAPP_BUILD_TAG=vps-lab-001
  for service in auth user mail chat todo workschedule canteen payment gateway; do
    docker buildx build --platform linux/amd64 --load \
      --file docker/node-service.Dockerfile \
      --build-arg SERVICE_DIR="$service" \
      --tag "nrapp/$service:$NRAPP_BUILD_TAG" .
  done
)
```

Mỗi service phải build thành công; lỗi ở đâu xử lý ở đó. Trên máy Apple Silicon/ARM, build amd64 qua emulation có thể chậm. Không gửi image ARM lên VPS amd64 và kỳ vọng chạy được. `NODE_OPTIONS` runtime trong Compose không ảnh hưởng build stage; không ép compiler chạy với heap 128 MB.

Đóng gói toàn bộ tag trong một archive để Docker dùng chung layer, không cần registry trả phí:

```bash
(
  set -euo pipefail
  NRAPP_BUILD_TAG=vps-lab-001
  NRAPP_TAGS=()
  for service in gateway auth user mail chat todo workschedule canteen payment; do
    NRAPP_TAGS+=("nrapp/$service:$NRAPP_BUILD_TAG")
  done
  docker image save "${NRAPP_TAGS[@]}" | gzip > /tmp/nrapp-images-vps-lab-001.tar.gz
  cd /tmp
  sha256sum nrapp-images-vps-lab-001.tar.gz > nrapp-images-vps-lab-001.sha256
)
scp -i ~/.ssh/nrapp_vps \
  /tmp/nrapp-images-vps-lab-001.tar.gz /tmp/nrapp-images-vps-lab-001.sha256 \
  deploy@203.0.113.10:/opt/nrapp/
```

Source vẫn đưa lên VPS theo bước 6 để có Compose/config/scripts. Không copy `node_modules` hoặc chạy watch. **VPS**, kiểm tra còn đủ đĩa cho cả archive lẫn image giải nén rồi import:

```bash
cd /opt/nrapp
sha256sum -c nrapp-images-vps-lab-001.sha256
(
  set -euo pipefail
  gzip -dc nrapp-images-vps-lab-001.tar.gz | docker image load
)
docker image ls --filter 'reference=nrapp/*:vps-lab-001'
```

Phải đủ 9 tag, root `backend/.env` có `NRAPP_IMAGE_TAG=vps-lab-001`. Chỉ sau checksum/import thành công và đủ tag mới xóa **hai file archive/checksum truyền tạm này** để trả dung lượng VPS; giữ bản gốc ở máy cá nhân:

```bash
rm /opt/nrapp/nrapp-images-vps-lab-001.tar.gz /opt/nrapp/nrapp-images-vps-lab-001.sha256
cd /opt/nrapp/backend
dc pull redis rabbitmq payment-postgres
mc pull
```

`mc pull` chỉ tải 3 image giám sát. Không chạy full logger wrapper. Nếu tag image từ repo không tồn tại trên registry, kiểm tra release chính thức trước khi đổi tag; không thay hàng loạt bằng `latest`.

### 10.2. Tạo network, bật hạ tầng và giám sát tối giản

Cả backend và minimal monitoring tham gia network external cùng tên. Tạo một lần; thao tác này không khởi động container:

```bash
cd /opt/nrapp/backend
docker network inspect nrapp-observability >/dev/null 2>&1 || docker network create nrapp-observability
dc up -d --wait --wait-timeout 300 redis rabbitmq payment-postgres
mc up -d --wait --wait-timeout 300
dc ps
mc ps
```

`mc ps` phải đúng 3 container. Minimal Prometheus chỉ có 2 scrape target, không có app probes nên không chờ dashboard readiness 9 app tự xuất hiện.

Kiểm tra hạ tầng:

```bash
dc exec -T redis redis-cli ping
dc exec -T redis redis-cli CONFIG GET maxmemory maxmemory-policy
dc exec -T rabbitmq rabbitmq-diagnostics -q ping
dc exec -T rabbitmq rabbitmq-diagnostics alarms
dc exec -T payment-postgres sh -c 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
dc exec -T payment-postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SHOW shared_buffers; SHOW max_connections;"'
```

Kỳ vọng Redis `PONG`, `noeviction` và maxmemory 50331648 byte, RabbitMQ ping thành công/không có alarm, PostgreSQL 64 MB shared_buffers/30 connections. Nếu RabbitMQ đã báo memory alarm ngay khi rảnh, 384 MiB có thể quá chật với runtime/queue hiện tại; đo memory breakdown và phân bổ lại trước khi khởi động app.

### 10.3. Nếu bắt buộc build trực tiếp trên VPS

Đây là phương án phụ khi máy cá nhân không build được. Dùng cửa sổ bảo trì, build từng service, không build đồng thời với đầy đủ app/monitoring đang chạy. Lần đầu dựng trước khi có dữ liệu có thể build trước bước 10.2; đã có app thì backup theo bước 14 và dừng app/monitoring tạm thời:

```bash
cd /opt/nrapp/backend
dc --profile app stop gateway auth user mail chat todo workschedule canteen payment
mc stop
(
  set -euo pipefail
  for service in auth user mail chat todo workschedule canteen payment gateway; do
    dc --profile app build "$service"
  done
)
```

`mem_limit` runtime không giới hạn quá trình BuildKit build. Có swap vẫn có thể OOM; đừng lặp lại build thất bại khi RAM không đủ. Nếu cần, dùng máy build khác rồi import như bước 10.1. Sau build thành công bật lại `mc up -d --wait`, app theo 10.4–10.5; nếu build fail vẫn phải khôi phục release cũ theo bước 15 để không bỏ hệ thống ở trạng thái dừng.

Kiểm tra disk/build cache bằng `docker system df`; chỉ xóa cache đã xác nhận không cần và giữ image rollback. Không xóa volume DB để lấy chỗ.

### 10.4. Bật User và Mail trước Auth

```bash
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 user mail
curl --fail --silent --show-error http://127.0.0.1:5000/health/ready
curl --fail --silent --show-error http://127.0.0.1:5001/health/ready
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 auth
curl --fail --silent --show-error http://127.0.0.1:4000/health
```

Mail không healthy: kiểm tra SMTP và biến uppercase trước khi thử lại login nhiều lần. Auth green chưa chứng minh Redis/RabbitMQ/MongoDB đang hoạt động nghiệp vụ.

### 10.5. Bật các service còn lại và Gateway

```bash
(
  set -e
  for service in chat todo workschedule canteen payment; do
    dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 "$service"
    docker stats --no-stream
    free -m
  done
)
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 gateway
dc --profile app ps
```

Kiểm tra readiness các app có route tương ứng:

```bash
for port in 5000 5001 5002 5003 5004 5005 5006; do
  printf '\nPort %s: ' "$port"
  curl --fail --silent --show-error --max-time 15 "http://127.0.0.1:$port/health/ready" || break
done
curl --fail --silent --show-error http://127.0.0.1:3000/health
```

Nếu fail, xem log service tương ứng; không tiếp tục bước public trước khi lỗi nền tảng được giải quyết.

### 10.6. Xác nhận migration Payment

Compose đang đặt `PAYMENT_DB_RUN_MIGRATIONS=true`; source dùng `synchronize:false` và tự chạy hai migration đã khai báo.

```bash
dc exec -T payment-postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT id, name FROM payment_migrations ORDER BY id;"'
```

Kỳ vọng thấy hai migration `InitialPaymentSchema1724000000000`, `AddOutboxObservability1724000001000` đã áp dụng. SQL tránh chạy thêm một Node/TypeORM CLI trong container Payment đang bị giới hạn RAM. Không gọi script migration có build trong runtime image đã bỏ dev dependencies; mọi migration mới phải được kiểm thử trước trên máy build/lab.

### 10.7. Kiểm tra MongoDB và index

Sau khi container User hoạt động:

```bash
dc exec -T user node -e 'const m=require("mongoose"); m.connect(process.env.MONGO_URL,{dbName:"nrapp"}).then(async()=>{await m.connection.db.command({ping:1}); console.log("MongoDB nrapp: OK"); await m.disconnect();}).catch(e=>{console.error(e.name);process.exit(1);});'
```

Kết nối thành công chưa kiểm tra transaction; thực hiện xuất kho/duyệt lịch ở bước 12. Kiểm tra index bằng Compass/mongosh trên đúng `nrapp`; Mongoose schema không tự loại bỏ index cũ. Đối chiếu [Canteen database indexes](../backend/canteen/docs/database-indexes.md); không chạy `dropIndexes()` hoặc `syncIndexes()` hàng loạt.

**Hoàn thành bước 10 khi:** 12 container backend ở trạng thái mong đợi, 9 app có HTTP kiểm tra tương ứng, migration Payment đúng và MongoDB truy cập được.

## 11. DNS, Nginx và HTTPS

### 11.1. Trỏ domain

1. Tạo DNS record `A`: `api` → IPv4 VPS.
2. Nếu dùng Cloudflare quản lý DNS, chọn **DNS only** trong bài đầu. Cloudflare chỉ phân giải DNS, request đi trực tiếp đến Nginx; cấu hình trust proxy ở bước 9 phù hợp đường đi này.
3. Không tạo record `AAAA` nếu chưa cấu hình IPv6 đúng trên VPS và firewall.
4. **Máy cá nhân**, kiểm tra:

```bash
dig +short A api.example.com
dig +short AAAA api.example.com
```

Record A phải trả IP thật của VPS. Nếu có AAAA cũ trỏ nơi khác, sửa/xóa record trước khi cấp certificate.

Chưa có domain: tạm kiểm tra qua `ssh -L 13000:127.0.0.1:3000 deploy@IP`, mở `http://127.0.0.1:13000/api-docs`. Hoàn thành domain/HTTPS trước khi cấu hình webhook ngoài Internet.

### 11.2. Tạo Nginx reverse proxy

**VPS**, tạo file `/etc/nginx/sites-available/nrapp-api`:

```bash
sudo nano /etc/nginx/sites-available/nrapp-api
```

Thay `api.example.com` bằng domain thật. Cấu hình dưới đây dùng một `location /` chung nên REST và `/socket.io/` đều giữ nguyên path qua Gateway:

```nginx
map $http_upgrade $nrapp_connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    server_name api.example.com;

    client_max_body_size 10m;

    # Swagger xem qua SSH tunnel vào Gateway khi quản trị.
    location ^~ /api-docs {
        return 404;
    }

    # Bài lab dùng email/OTP. Mở lại sau khi xác minh Google token đầy đủ.
    location = /api/auth/login-google {
        return 404;
    }
    location = /api/auth/login-google/ {
        return 404;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $nrapp_connection_upgrade;
        proxy_connect_timeout 5s;
        proxy_read_timeout 75s;
        proxy_send_timeout 75s;
    }
}
```

Trên Ubuntu, `sites-enabled/*` nằm trong context `http`, nên `map` ở đầu file này hợp lệ. `proxy_pass` không thêm `/api` và không strip `/socket.io`. Không thêm header CORS ở Nginx vì Gateway đã xử lý CORS.

Nginx phải truyền `Upgrade`/`Connection` để WebSocket upgrade đi qua; timeout cần đủ cho kết nối realtime. [Nginx WebSocket proxying](https://nginx.org/en/docs/http/websocket.html).

`client_max_body_size 10m` là giới hạn Nginx; giới hạn payload thực còn phụ thuộc body parser/upload của ứng dụng. Gặp 413 phải xác định lớp nào từ chối, không chỉ tăng Nginx.

Kích hoạt:

```bash
sudo ln -s /etc/nginx/sites-available/nrapp-api /etc/nginx/sites-enabled/nrapp-api
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl reload nginx
curl --fail --silent --show-error -H 'Host: api.example.com' http://127.0.0.1/health
```

Nếu symlink đã có thì không tạo lần nữa. `nginx -t` phải thành công rồi mới reload.

### 11.3. Cấp certificate Let's Encrypt

Thực hiện sau khi DNS đúng, inbound 80/443 mở và Nginx phục vụ domain. **VPS sạch chưa cài Certbot bằng apt**, dùng cách snap theo [Certbot cho Nginx](https://certbot.eff.org/instructions?ws=nginx&os=snap):

```bash
sudo apt install -y snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
sudo certbot --nginx -d api.example.com
```

Nhập email quản trị thật, đọc điều khoản và bật chuyển HTTP sang HTTPS khi được hỏi. Nếu Certbot/symlink đã có, kiểm tra `command -v certbot` và cách cài hiện tại trước, không cài chồng hai phương thức.

Kiểm tra:

```bash
sudo nginx -t
curl --fail --silent --show-error https://api.example.com/health
curl -I http://api.example.com/health
sudo certbot certificates
sudo certbot renew --dry-run
systemctl list-timers --all
```

Mong đợi HTTPS trả 200 với certificate hợp lệ, HTTP redirect sang HTTPS, dry-run thành công. Với snap, kiểm tra thêm `snap services certbot` nếu không thấy timer tên Certbot trong systemd. Không dùng `curl -k` để chấp nhận certificate sai trong bài nghiệm thu.

### 11.4. Kiểm tra cổng từ ngoài VPS

**VPS**:

```bash
sudo ss -lntp
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

**Máy cá nhân**, thay IP thật:

```bash
curl --fail --silent --show-error https://api.example.com/health
nc -vz -w 3 203.0.113.10 3000
nc -vz -w 3 203.0.113.10 5002
nc -vz -w 3 203.0.113.10 5433
nc -vz -w 3 203.0.113.10 6379
nc -vz -w 3 203.0.113.10 15672
nc -vz -w 3 203.0.113.10 3001
```

HTTPS phải được; các port nội bộ trên phải refused/timeout. Nếu port nội bộ kết nối được, kiểm tra bind IP, effective Compose, cloud firewall trước khi tiếp tục. Khi có IPv6, kiểm tra cả địa chỉ IPv6 public.

### 11.5. CORS và Google login trước khi có người dùng thật

Gateway hiện hardcode `origin: '*'`, `credentials:false`. Tài liệu không giả định có biến `CORS_ORIGINS` vì source chưa đọc biến này. Với lab Bearer token, thử origin frontend thực tế để xác nhận. Trước khi dùng public lâu dài, bổ sung allowlist origin trong source và test preflight; cookie auth cần một thiết kế CORS/cookie riêng.

Hàm Google login hiện gọi Google tokeninfo/userinfo nhưng chưa thấy kiểm tra audience client ID trong code đã đọc. Nginx mẫu chặn path phổ biến để bài học dùng email/OTP; đây **không thay thế** sửa xác minh token hoặc vô hiệu hóa route ở ứng dụng. Trước khi mở Google login, cần kiểm tra `aud`, issuer, hạn token và trạng thái email theo hợp đồng OAuth của ứng dụng. Nếu cần đảm bảo route bị vô hiệu hoàn toàn, chặn ở controller/feature flag và test các biến thể route, thay vì chỉ dựa vào Nginx.

## 12. Kiểm tra từng luồng nghiệp vụ

### 12.1. Bộ dữ liệu thử và công cụ

Chuẩn bị hai email bạn sở hữu để nhận OTP, một tài khoản sẽ làm admin lab và một user thường. Dùng Postman hoặc Swagger qua tunnel để không đưa mật khẩu/token vào shell history.

**Máy cá nhân**, giữ terminal này mở:

```bash
ssh -i ~/.ssh/nrapp_vps -N \
  -L 13000:127.0.0.1:3000 \
  -L 13001:127.0.0.1:3001 \
  -L 15672:127.0.0.1:15672 \
  deploy@203.0.113.10
```

Mở `http://127.0.0.1:13000/api-docs` để đọc DTO/route và test nội bộ. Sau đó lặp lại các luồng chính bằng Postman với base URL **`https://api.example.com`** để kiểm tra cả DNS/TLS/Nginx. Tunnel bỏ qua Nginx nên chỉ test tunnel chưa đủ.

### 12.2. Auth → Redis → RabbitMQ → Mail → User

Thực hiện đúng thứ tự; theo source, **register chưa gửi OTP, login mới gửi**:

| Bước | Request qua Gateway | Kết quả cần thấy |
|---|---|---|
| 1 | `POST /api/auth/register` với `email`, `password`, `username` | Trả `userId`; record credential xuất hiện |
| 2 | Kiểm tra User/RabbitMQ | Profile `users` có cùng ID sau consumer `user-profile-sync` xử lý |
| 3 | `POST /api/auth/login` với `email`, `password` | API báo gửi OTP; inbox nhận email |
| 4 | `POST /api/auth/verify` với `email`, `otp` | Nhận access token và refresh token theo response thật |
| 5 | `GET /api/auth/me` với `Authorization: Bearer ACCESS_TOKEN` | Đúng tài khoản/profile; chữ ký Gateway → Auth hợp lệ |
| 6 | `POST /api/auth/refresh` với trường **`refreshToken`** | Có session/token mới; không gửi trường `token` thay thế |
| 7 | API cần đăng nhập, bỏ token | Bị từ chối; không trả dữ liệu của user |
| 8 | Tài khoản user gọi API admin | Bị từ chối theo role |

Ví dụ payload để dán vào Postman, thay dữ liệu của bạn:

```json
{
  "email": "your-lab-email@example.com",
  "password": "REPLACE_WITH_A_UNIQUE_LAB_PASSWORD",
  "username": "vps-lab-user"
}
```

Login có giới hạn gửi OTP một lần mỗi 60 giây; OTP hết hạn sau 5 phút; sai nhiều lần bị khóa theo code. Không test tải bằng login/OTP. Register thành công nhưng profile chưa có: xem queue/consumer; source có nhánh bắt lỗi publish nên HTTP thành công chưa bảo đảm đồng bộ profile hoàn tất.

### 12.3. Tạo admin đầu tiên trong cluster lab

Chưa thấy script seed admin đi kèm backend. Với **cluster lab mới**, có thể bootstrap một tài khoản đã đăng ký bằng giao diện Atlas/Compass:

1. Đăng ký email admin lab qua API như trên.
2. Chờ profile tồn tại trong `users`; đối chiếu `_id` với credential.
3. Trên đúng database `nrapp`, tìm document email đó trong collection `credentials` theo schema hiện tại, đổi **duy nhất** `role` thành `admin`.
4. Tìm profile cùng `_id` trong `users`, đổi `role` thành `admin` để đồng bộ dữ liệu ban đầu.
5. Đăng nhập/verify lại để nhận token mới; gọi API admin và kiểm tra tài khoản user thường vẫn bị cấm.
6. Với tài khoản tiếp theo, dùng API quản lý role `/api/auth/users/:userId/role` để có luồng đồng bộ RabbitMQ; không duy trì cách sửa DB thủ công.

Kiểm tra đúng collection thực tế trước khi sửa, không sửa hàng loạt theo filter rỗng. Thao tác bootstrap này chỉ dành cho tài khoản lab đầu tiên; nếu đưa vào vận hành cần script bootstrap có kiểm soát và ghi nhận người thực hiện.

### 12.4. Chat HTTP, Socket.IO và ảnh

1. Đăng nhập hai user trong hai trình duyệt/thiết bị.
2. Cấu hình app dùng API domain; Socket.IO dùng origin **`https://api.example.com`**, path **`/socket.io`**. Không trỏ client vào port 5002 hoặc `http://chat:5002`.
3. Chat gateway nhận token tại `socket.handshake.auth.token`; không gửi refresh token hoặc mặc định rằng query string token được hỗ trợ.
4. Gửi text A → B, kiểm tra B nhận ngay không cần refresh, trạng thái seen và typing hoạt động.
5. Upload ảnh nhỏ, kiểm tra API upload, message và tài nguyên Cloudinary.
6. Đóng/mở lại mạng, xem client reconnect và trạng thái online cập nhật.
7. Thử token sai/hết hạn: Socket.IO connection phải bị từ chối.

Kiểm tra Engine.IO polling qua **máy cá nhân**:

```bash
curl --fail --silent --show-error 'https://api.example.com/socket.io/?EIO=4&transport=polling'
```

Mong đợi gói mở Engine.IO thường bắt đầu bằng `0{...}`. Đây chưa phải đăng nhập Socket.IO thành công. Dùng DevTools → Network → WS xác nhận upgrade **101** khi client dùng WebSocket, rồi kiểm tra message thật.

### 12.5. Todo, Workschedule, Canteen

Đọc route/DTO ở Swagger đang chạy, tránh đoán tên endpoint của các module. Ghi ID dữ liệu thử để đối chiếu sau restore.

| Module | Bài thực hành | Điều kiện đạt |
|---|---|---|
| User | Xem profile, đổi tên, thử quyền quản trị | Auth/User nhất quán theo luồng hiện có |
| Todo | Tạo, đọc, sửa, hoàn thành, xóa một việc của user lab | Dữ liệu lưu MongoDB; tài khoản khác không đọc/sửa trái quyền |
| Workschedule | Tạo yêu cầu/lịch theo vai trò, duyệt, đọc lại, thử chấm công | Giao dịch duyệt lịch không báo lỗi standalone transaction |
| Canteen danh mục | Admin tạo nguyên liệu, danh mục, món, bàn | User xem được catalog đúng dữ liệu |
| Canteen kho | Nhập hai lô, xuất một lượng qua API xuất kho | FEFO/số lượng nhất quán; transaction thành công |
| Canteen đơn/bếp | Tạo đơn, nhận xử lý ở bếp, hoàn thành theo trạng thái hợp lệ | Đơn, tồn kho/bàn và quyền từng vai trò đúng |
| Phân quyền | User gọi thao tác admin/chef/manager không được cấp | Bị từ chối; không chỉ ẩn nút trên UI |

Các bài kiểm tra ghi dữ liệu phải dùng cluster/database lab. Không coi `npm test` hoặc healthcheck là thay thế các bước này.

### 12.6. Payment — chia ba mức kiểm tra

**Mức 1: hệ thống và xác thực, chưa cần chuyển tiền.** Health Payment xanh, migration đã áp dụng, request webhook không có chữ ký bị từ chối:

```bash
curl -i -X POST https://api.example.com/api/payment/webhooks/casso \
  -H 'Content-Type: application/json' \
  --data '{}'
```

Kỳ vọng phản hồi 4xx xác thực, thường 401 theo handler chữ ký hiện tại. Không kỳ vọng 200 khi thiếu secret/signature; không được sửa code bỏ xác thực để làm test xanh.

**Mức 2: luồng ứng dụng và webhook giả lập trên dữ liệu lab.**

1. Tạo đơn Canteen thuộc user lab với `paymentMethod=VIETQR` theo DTO.
2. Gọi `POST /api/payment/create-qr` với `orderId` qua Gateway và access token.
3. Kiểm tra QR, số tiền và tài khoản. Gateway đọc `finalAmount` từ Canteen rồi ký request sang Payment; client không tự quyết định amount.
4. Gọi API đọc giao dịch để thấy `PENDING`.
5. Test script có sẵn `payment/scripts/payment-smoke.mjs` tạo payment và webhook giả, **có ghi dữ liệu**, gọi trực tiếp Payment port 5006. Chỉ chạy khi dữ liệu/secret đều là lab, và hiểu nó không chứng minh tuyến Casso thật hoặc Gateway → Canteen tạo QR đúng.

**VPS**, sau khi đồng bộ `PAYMENT_INTERNAL_SECRET` trong `payment/.env` với root:

```bash
cd /opt/nrapp/backend/payment
npm run test:smoke
```

Script chạy bằng Node 22 host, nạp env service/root theo package script. Giữ `PAYMENT_SMOKE_BASE_URL` mặc định `http://127.0.0.1:5006`; không đổi sang Gateway vì hai endpoint có hợp đồng xác thực khác. Script tạo order ID giả; sự kiện tới Canteen có thể không tìm thấy đơn tương ứng, phải xem log/queue. Dùng test qua đơn thật của lab để kiểm tra settlement end-to-end.

**Mức 3: webhook từ nhà cung cấp thực tế.**

1. Đặt callback Casso tới URL HTTPS ở bước 7, chọn đúng V2/chữ ký tương ứng code.
2. Thực hiện chức năng test của nhà cung cấp nếu có và hợp đồng payload tương thích.
3. Nếu chủ động thử giao dịch thực, dùng tài khoản của bạn và giá trị nhỏ phù hợp chính sách nhà cung cấp; đối chiếu tiền thực nhận, không dựa riêng trạng thái UI.
4. Theo dõi callback → Payment PostgreSQL → outbox → RabbitMQ → Canteen cập nhật trạng thái đơn.
5. Kiểm tra webhook gửi lại không tạo thanh toán trùng; sai tài khoản/số tiền không tự đánh dấu đơn đã trả tiền.

Giữ `CASSO_SIGNATURE_MAX_AGE_MS` theo chủ ý hiện tại và hợp đồng retry. Source mẫu để `0` nhằm chấp nhận retry có chữ ký; idempotency ở PostgreSQL vẫn quan trọng. Không xoay secret Casso ngẫu nhiên sau mỗi deploy.

### 12.7. Cập nhật cấu hình client

- Web/mobile gọi **Gateway HTTPS** cho API; đường base có `/api` hay không phải khớp cách client nối path, tránh `/api/api/...`.
- Socket.IO dùng cùng domain theo bước 12.4.
- Thay URL LAN/localhost trong cấu hình client rồi rebuild nếu biến được đóng gói lúc build.
- Với native app, tạo lại bản build khi cần; điện thoại thật không nhìn thấy `localhost` của VPS.
- Deploy web frontend là bài tiếp theo; để kiểm tra backend có thể chạy web local trỏ API VPS. Đối chiếu rule `CLAUDE.md` riêng của frontend trước khi sửa frontend.

## 13. Giám sát và cảnh báo

### 13.1. Nghiệm thu minimal monitoring thay bộ smoke đầy đủ

**Không chạy** `npm run observability:up`, `observability:smoke`, `observability:acceptance` hoặc wrapper `scripts/observability-compose.mjs` trên lộ trình này. Các lệnh đó dành cho full stack: có lệnh bật lại toàn bộ, có lệnh đòi Collector/Loki/Jaeger/Alloy và chắc chắn không phản ánh đúng phạm vi minimal.

**VPS**, kiểm tra đúng 3 service và hai target:

```bash
mc config --services
mc ps
curl --fail --silent --show-error http://127.0.0.1:9090/-/ready
curl --fail --silent --show-error http://127.0.0.1:3001/api/health
curl --fail --silent --show-error http://127.0.0.1:9090/api/v1/targets \
  | jq -r '.data.activeTargets[] | [.labels.job, .health, .lastError] | @tsv'
curl --fail --silent --show-error --get http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=node_memory_MemAvailable_bytes' \
  | jq '.data.result'
```

Sau tối đa vài chu kỳ scrape 60 giây, chỉ thấy jobs `prometheus` và `node-exporter`, đều `up`; query memory phải có giá trị. Query rỗng dù target up: kiểm tra node-exporter mounts, quyền và metric thật. Không chờ `application-metrics`/`backend-readiness` vì chúng không được cấu hình trong minimal.

Kiểm tra app health riêng bằng các lệnh bước 10 và nghiệp vụ bước 12. Prometheus `up=1` cho Node Exporter chỉ chứng minh scrape host được, không chứng minh API/DB/OTP đang tốt.

### 13.2. Dashboard CPU/RAM/disk đủ dùng

**Máy cá nhân**, thêm tunnel Prometheus nếu muốn xem UI trực tiếp:

```bash
ssh -i ~/.ssh/nrapp_vps -N \
  -L 19090:127.0.0.1:9090 \
  deploy@203.0.113.10
```

Dùng tunnel Grafana/RabbitMQ đã có ở bước 12:

| Công cụ | URL trên máy cá nhân | Phạm vi |
|---|---|---|
| Grafana | `http://127.0.0.1:13001` | Dashboard host từ Prometheus |
| Prometheus | `http://127.0.0.1:19090` | Targets, PromQL; không mở public |
| RabbitMQ UI | `http://127.0.0.1:15672` | Xem consumers, ready/unacked, DLQ thủ công |

Trong Grafana:

1. Đăng nhập bằng tài khoản trong `logger/.env`.
2. Kiểm tra datasource **Prometheus** đã được provision, URL container `http://prometheus:9090`.
3. Tạo dashboard mới, thêm từng visualization với datasource này; chọn refresh 60 giây.
4. Dùng các query dưới đây, lưu tên `VPS 4GB - Host overview`.
5. Kiểm tra đồ thị RAM khớp gần với `free -m`, disk khớp filesystem thật. Không import dashboard full repo có dependency vào cAdvisor/Loki/Jaeger.

| Panel | PromQL | Unit |
|---|---|---|
| CPU sử dụng | `100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))` | Percent 0–100 |
| RAM sử dụng | `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` | Percent 0–100 |
| RAM còn sẵn | `node_memory_MemAvailable_bytes / 1024 / 1024` | MiB |
| Disk root sử dụng | `100 * (1 - node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs\|overlay"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs\|overlay"})` | Percent 0–100 |
| Swap đang dùng | `(node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / 1024 / 1024` | MiB |

Dấu `\|` ở hàng disk chỉ để escape bảng Markdown; query dán vào Grafana dùng regex `tmpfs|overlay` (không có dấu gạch chéo trước `|`). Nếu root mountpoint khác do cấu hình host, xem labels `node_filesystem_size_bytes` rồi chọn đúng mountpoint. RSS từng container xem `docker stats`; Node Exporter không thay cAdvisor.

### 13.3. Log, health và tín hiệu mất đi khi cắt stack

**VPS**, xem log theo service hoặc request ID:

```bash
dc logs --tail=100 gateway auth user mail
dc logs --since=15m payment canteen
dc logs --since=15m gateway auth | rg --fixed-strings 'REPLACE_REQUEST_ID'
mc logs --tail=100 prometheus grafana node-exporter
docker stats --no-stream
```

Giữ JSON structured log, error ID/request ID vẫn hỗ trợ tìm lỗi theo source hiện tại. Không hứa có trace liên dịch vụ hoặc tìm log lịch sử trên Grafana: SDK đã tắt, không có Collector/Jaeger/Loki/Alloy. Log driver local quay vòng nên log cũ sẽ bị loại; xuất đoạn cần điều tra trước khi rotate. Logger package vẫn được build/import vì app phụ thuộc nó.

Minimal cũng không có dashboard p95/error rate của API, history queue/DB/container metrics hoặc probe liên tục 9 app. Thay bằng healthcheck Docker + kiểm tra nghiệp vụ thủ công, Nginx log và RabbitMQ UI trong giai đoạn đầu. Nếu cần lịch sử này sau khi có tải thật, bổ sung có mục tiêu và đo lại RAM.

Không triển khai kênh cảnh báo ngoài trong cấu hình đầu tiên để giữ bước học đơn giản. Có thể tạo Grafana alert theo metric host ở giai đoạn sau, nhưng phải test tải/quota và thông báo; **hiện tại không có cam kết nhận cảnh báo khi VPS chết**. Alert đặt trên cùng VPS cũng không bảo đảm báo được khi cả máy mất kết nối.

### 13.4. Đo RAM và quyết định giữ gói 4 GB

Sau mỗi nhóm app ở bước 10.4–10.5, rồi sau khi đủ 15 container:

```bash
free -m
docker stats --no-stream
vmstat 1 10
sudo journalctl -k --since '1 hour ago' --no-pager
for service in gateway auth user mail chat todo workschedule canteen payment; do
  NRAPP_CONTAINER_ID=$(dc ps -q "$service")
  test -n "$NRAPP_CONTAINER_ID" || continue
  docker inspect --format '{{.Name}} memory_limit={{.HostConfig.Memory}} oom={{.State.OOMKilled}} restarts={{.RestartCount}}' "$NRAPP_CONTAINER_ID"
done
```

`memory_limit` là bytes; `OOMKilled`/restart count là dấu hiệu cần đọc cùng log/kernel, không coi một snapshot false là chứng minh chưa từng có OOM. Lỗi Node `JavaScript heap out of memory` khác container bị kernel kill; cả hai cần xử lý. Đo cả lúc boot, login/bcrypt, upload ảnh, transaction và healthcheck chạy chồng, không chỉ trạng thái idle.

Bài nghiệm thu bắt buộc trước khi giữ gói thuê lâu dài:

1. Khởi động đủ **12 backend + 3 monitoring**; nếu app không boot với limit mẫu, đo và điều chỉnh ngay.
2. Để chạy rảnh 15 phút, ghi RAM/CPU/queue baseline.
3. Dùng hai tài khoản thử luân phiên các luồng bước 12 trong 30–60 phút; không dùng request OTP lặp liên tục làm benchmark.
4. Dành một chu kỳ 24 giờ tải lab bình thường để phát hiện restart/heap tăng/disk growth; xem `vmstat` cột `si/so` có liên tục tăng không.
5. Mục tiêu khởi đầu: không có OOM/restart lặp, không RabbitMQ memory alarm kéo dài, `MemAvailable` còn khoảng **500 MiB trở lên** lúc tải lab ổn định, không swap-in/out liên tục, còn từ 10 GB disk. Đây là tiêu chí chọn để có khoảng trống vận hành, không phải số đo đã đạt.
6. Nếu chưa đạt: xác nhận không còn full logger/pgAdmin/app host; giảm concurrency nhập liệu/upload, kiểm tra queue bị kẹt và connection pool; build/backup ở thời điểm ít tải. Nới đúng service thiếu bộ nhớ trong ngân sách tổng hoặc tối ưu source bằng một task riêng có số đo.
7. Nếu RSS cần thiết của đủ 9 app + 3 hạ tầng + 3 monitoring và OS vẫn vượt khả năng máy, **4 GB không đủ cho workload đó**. Giữ đủ 9 service như yêu cầu; khi đó cần tối ưu thêm hoặc tăng RAM, không âm thầm bỏ service và không khẳng định swap đã giải quyết.

Ngưỡng từng container dùng trên 80–85% limit thường xuyên là tín hiệu xem xét, không tự động tăng limit toàn bộ. Tổng hard limit 3280 MiB không bao gồm container `docker run` tạm khi backup hoặc tiến trình build; không chạy các bài đó chồng giờ tải.

## 14. Backup và diễn tập restore

### 14.1. Chọn dữ liệu cần giữ

| Dữ liệu | Bản sao cần có |
|---|---|
| PostgreSQL Payment | `pg_dump -Fc`, xác nhận `pg_restore` đọc/restore được |
| MongoDB `nrapp` | Atlas backup ở tier hỗ trợ, hoặc `mongodump` cho lab và diễn tập `mongorestore` |
| Env/secrets/cấu hình | Bản sao mã hóa ngoài VPS; gồm Compose minimal, Prometheus/Grafana provisioning và RabbitMQ VPS config |
| Redis | Xác định chấp nhận mất phiên/OTP hay cần lưu RDB/AOF nhất quán |
| RabbitMQ | Definitions để dựng lại topology; messages cần chiến lược riêng, definitions không chứa message |
| Source/image release | Commit/tag hoặc bản source đóng gói và image đã chạy tốt |
| Cloudinary | Danh sách/tài nguyên media và cơ chế backup của tài khoản; backup DB không chứa file ảnh |

Mục tiêu ban đầu cho lab: backup sau buổi thực hành và trước mỗi deploy; đặt mục tiêu mất tối đa dữ liệu từ lần backup gần nhất (**RPO**) và khôi phục trong khoảng 1–2 giờ (**RTO**) rồi đo lại. Đây là mục tiêu cần diễn tập, chưa phải năng lực đã xác nhận.

**Một file backup chỉ nằm trên cùng VPS chưa bảo vệ khỏi mất VPS.** Sau khi kiểm tra, copy bản backup mã hóa sang máy cá nhân hoặc nơi lưu trữ ngoài VPS.

### 14.2. Chuẩn bị MongoDB Database Tools

Để không cài thêm công cụ DB lên host, bài lab dùng container công cụ `mongo:8.0`. Đây là container **chạy tạm để dump/restore**, không phải MongoDB server thứ 16 chạy thường trực. Chạy ngoài giờ tải, ưu tiên lúc các app đã dừng theo 14.3; nếu disk không đủ image tool, chạy công cụ từ máy cá nhân đã allowlist Atlas. Kiểm tra image chứa các tool trước khi dùng:

```bash
docker pull mongo:8.0
docker run --rm mongo:8.0 mongodump --version
docker run --rm mongo:8.0 mongorestore --version
```

Nếu image/tag dùng tại lúc thực hành không có tool, cài MongoDB Database Tools từ gói chính thức tương thích hệ điều hành; không bỏ qua bước restore vì thiếu tool.

Tạo file `/home/deploy/.config/nrapp/mongodump.yml` chứa **URI Atlas lab** và bảo vệ quyền file:

```bash
mkdir -p /home/deploy/.config/nrapp
chmod 700 /home/deploy/.config/nrapp
umask 077
nano /home/deploy/.config/nrapp/mongodump.yml
chmod 600 /home/deploy/.config/nrapp/mongodump.yml
```

Nội dung YAML, dùng credential có quyền đọc `nrapp`:

```yaml
uri: "mongodb+srv://nrapp_lab:REPLACE_PASSWORD@REPLACE_CLUSTER.mongodb.net/nrapp?retryWrites=true&w=majority"
```

Không đưa URI thật vào command line; bind file cấu hình cho công cụ. `mongodump` hỗ trợ file config cho thông tin nhạy cảm. [MongoDB Database Tools: mongodump](https://www.mongodb.com/docs/database-tools/mongodump/).

### 14.3. Tạo cặp backup nhất quán cho bài lab

Payment/PostgreSQL và đơn Canteen/MongoDB liên quan nhau. Dump hai DB ở hai thời điểm khi vẫn có người ghi có thể tạo chênh lệch nghiệp vụ. Với lab ít người, dùng cửa sổ bảo trì ngắn:

1. Ngừng thao tác thử từ client, để worker xử lý xong và kiểm tra backlog RabbitMQ.
2. Dừng Gateway để không nhận thêm request mới; đang bật Casso thật thì phối hợp retry/đối soát của nhà cung cấp.
3. Dừng các app ghi DB/consumer/worker rồi dump cả hai database; không dừng Redis/RabbitMQ/PostgreSQL trong bài này.
4. Nếu còn queue pending/unacked, ghi nhận rõ; tốt nhất đưa về trạng thái yên trước khi lấy cặp backup. Không purge queue để làm số liệu đẹp.

**VPS**, chạy từng khối và kiểm tra lỗi. Khối sau dừng các app, Nginx sẽ tạm trả 502:

```bash
dc --profile app stop gateway
dc exec -T rabbitmq rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
dc --profile app stop auth user mail chat todo workschedule canteen payment
```

Tạo thư mục riêng cho lần backup trong **cùng phiên SSH**:

```bash
umask 077
NRAPP_BACKUP_DIR="/opt/nrapp-backups/$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$NRAPP_BACKUP_DIR"
printf 'Backup directory: %s\n' "$NRAPP_BACKUP_DIR"
dc exec -T payment-postgres sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' > "$NRAPP_BACKUP_DIR/payment.dump"
test -s "$NRAPP_BACKUP_DIR/payment.dump"
docker run --rm --user "$(id -u):$(id -g)" \
  --mount type=bind,src=/home/deploy/.config/nrapp/mongodump.yml,dst=/tmp/mongodump.yml,readonly \
  mongo:8.0 mongodump --config=/tmp/mongodump.yml --db=nrapp --archive --gzip --numParallelCollections=1 \
  > "$NRAPP_BACKUP_DIR/mongo-nrapp.archive.gz"
test -s "$NRAPP_BACKUP_DIR/mongo-nrapp.archive.gz"
dc exec -T payment-postgres pg_restore --list < "$NRAPP_BACKUP_DIR/payment.dump" > "$NRAPP_BACKUP_DIR/payment-contents.txt"
```

**Mỗi lệnh dump phải exit 0. File khác 0 byte chưa đủ chứng minh dump thành công.** Nếu một lệnh fail, không đánh dấu cặp backup thành công; giữ các file lỗi để điều tra hoặc lấy lại cặp backup trong cửa sổ mới. Sau khi hoàn tất hoặc quyết định dừng bài thử, bật app trở lại:

```bash
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 user mail
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 auth chat todo workschedule canteen payment
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 gateway
```

Lưu checksum **sau khi xác nhận dump thành công**:

```bash
cd "$NRAPP_BACKUP_DIR"
sha256sum payment.dump mongo-nrapp.archive.gz > SHA256SUMS
sha256sum -c SHA256SUMS
cd /opt/nrapp/backend
```

Tài khoản Atlas Free không hỗ trợ `mongodump --oplog`; bài lab tạm dừng các nguồn ghi để tránh dựa vào option không hỗ trợ. [Atlas Free: giới hạn công cụ và backup](https://www.mongodb.com/docs/atlas/reference/free-shared-limitations/).

### 14.4. Lưu cấu hình và copy ra ngoài

Từ **VPS**:

```bash
cd /opt/nrapp/backend
tar -czf "$NRAPP_BACKUP_DIR/config-private.tar.gz" \
  .env logger/.env gateway/.env auth/.env user/.env mail/.env chat/.env \
  todo/.env workschedule/.env canteen/.env payment/.env \
  compose.yaml compose.vps.yaml docker/rabbitmq.vps.conf \
  logger/compose.vps-minimal.yaml logger/vps-minimal
sudo cp /etc/nginx/sites-available/nrapp-api "$NRAPP_BACKUP_DIR/nginx-nrapp-api.conf"
sudo chown deploy:deploy "$NRAPP_BACKUP_DIR/nginx-nrapp-api.conf"
chmod 600 "$NRAPP_BACKUP_DIR"/*
```

Archive này chứa secret thật. Mã hóa cả thư mục backup bằng công cụ bạn quản lý khóa trước khi đưa lên nơi lưu trữ dùng chung. Với bản sao về máy cá nhân qua SSH, dùng disk mã hóa và lưu đúng quyền; không commit archive.

**Máy cá nhân**, thay `BACKUP_TIMESTAMP` bằng tên thư mục vừa in:

```bash
mkdir -p ~/nrapp-vps-backups
chmod 700 ~/nrapp-vps-backups
scp -i ~/.ssh/nrapp_vps -r \
  deploy@203.0.113.10:/opt/nrapp-backups/BACKUP_TIMESTAMP \
  ~/nrapp-vps-backups/
```

Sau khi copy, chạy `sha256sum -c SHA256SUMS` trong thư mục backup máy cá nhân. Xác nhận bản ngoài VPS đọc được trước khi xóa các bản quá hạn. Gợi ý giữ 7 bản ngày và 4 bản tuần tùy dung lượng; chưa tự chạy lệnh xóa volume/backup.

### 14.5. Restore PostgreSQL vào DB mới, không ghi đè DB đang chạy

**VPS**, làm lúc tải thấp/cửa sổ bảo trì để `pg_restore` không tranh RAM/connection với Payment, đặt biến tới một backup đã kiểm tra. Ví dụ:

```bash
NRAPP_BACKUP_DIR=/opt/nrapp-backups/BACKUP_TIMESTAMP
dc exec -T payment-postgres sh -c 'createdb -U "$POSTGRES_USER" nrapp_payment_restore_lab'
dc exec -T payment-postgres sh -c 'pg_restore --exit-on-error --no-owner --no-privileges -U "$POSTGRES_USER" -d nrapp_payment_restore_lab' < "$NRAPP_BACKUP_DIR/payment.dump"
dc exec -T payment-postgres sh -c 'psql -U "$POSTGRES_USER" -d nrapp_payment_restore_lab -c "SELECT id, name FROM payment_migrations ORDER BY id;"'
```

Chỉ tạo tên DB này nếu chưa tồn tại; nếu đã có từ lần thử trước, chọn tên restore lab mới. Không dùng `--clean` vào DB live. So sánh bảng, số lượng payment, webhook receipt/outbox và một số ID giao dịch đã ghi ở bước 12.

Để kiểm tra ứng dụng với DB restore, cần một môi trường thử riêng trỏ đúng DB restore, có RabbitMQ riêng hoặc vô hiệu worker được thiết kế rõ; không bật thêm Payment vào queue đang hoạt động chỉ để thử đọc dữ liệu.

### 14.6. Restore MongoDB vào cluster lab khác

1. Tạo cluster khôi phục riêng, user/allowlist riêng, database vẫn `nrapp` để tương thích source hardcode.
2. Tạo `/home/deploy/.config/nrapp/mongorestore.yml` tương tự file dump nhưng URI là **cluster restore**, không phải cluster đang chạy.
3. Dùng tài khoản đủ quyền restore database đích, bảo vệ file quyền 600.
4. Chạy trên VPS:

```bash
docker run --rm -i --user "$(id -u):$(id -g)" \
  --mount type=bind,src=/home/deploy/.config/nrapp/mongorestore.yml,dst=/tmp/mongorestore.yml,readonly \
  mongo:8.0 mongorestore --config=/tmp/mongorestore.yml \
  --archive --gzip --nsInclude='nrapp.*' --stopOnError --numParallelCollections=1 \
  < "$NRAPP_BACKUP_DIR/mongo-nrapp.archive.gz"
```

5. Kiểm tra collections/indexes và ID tài khoản/đơn/todo/lịch đã ghi trước đó.
6. Muốn nghiệm thu đủ end-to-end, dựng backend trên **VPS phục hồi riêng** theo hướng dẫn, dùng Mongo cluster restore + PostgreSQL restore + queue/Redis riêng + secret lab; không bật callback ngân hàng thật hoặc gửi OTP ngoài ý muốn.
7. Đăng nhập lại, xem dữ liệu, thử một thao tác ghi mới; đo thời gian từ VPS trống đến hệ thống hoạt động.

Restore thành công trên tool nhưng chưa đọc được bằng ứng dụng thì chưa hoàn thành bài phục hồi.

### 14.7. Redis, RabbitMQ và lịch tự động

Redis trong repo bật AOF và có volume. Nếu chấp nhận mất phiên/OTP khi phục hồi lab, ghi rõ chính sách buộc người dùng đăng nhập lại; không giả định PostgreSQL/Mongo dump sẽ khôi phục Redis. Nếu cần giữ, học snapshot nhất quán RDB/AOF, thử restore trên Redis riêng; không copy trực tiếp thư mục đang ghi rồi coi là backup đã kiểm chứng.

RabbitMQ definitions chỉ chứa topology/tài khoản/chính sách, không chứa message. Giữ definitions như sau nếu cần:

```bash
dc exec -T rabbitmq rabbitmqctl export_definitions /tmp/nrapp-definitions.json
dc cp rabbitmq:/tmp/nrapp-definitions.json "$NRAPP_BACKUP_DIR/rabbitmq-definitions.json"
chmod 600 "$NRAPP_BACKUP_DIR/rabbitmq-definitions.json"
```

Trong phục hồi lab, dựng broker mới và để app khai báo queue; nếu messages chưa được xử lý trước sự cố thì cần đối soát/replay có chủ đích. Không nối lại queue chứa message cũ với database đã lùi thời gian mà chưa đánh giá tác động, nhất là thanh toán.

Sau khi dump/restore thủ công đã thành công, mới tự động hóa:

1. Viết script backup có `set -euo pipefail`, đường dẫn tuyệt đối, khóa chống chạy chồng và mã lỗi khi dump/copy fail.
2. Với lab có cửa sổ dừng app, dùng trap/bước phục hồi bảo đảm app được bật lại nếu dump thất bại; kiểm tra health sau bật lại.
3. Đặt systemd timer/cron vào giờ không thực hành, ví dụ 02:00 giờ Việt Nam. Dùng `/home/deploy/bin/dc` vì cron không có PATH/NVM giống terminal.
4. Kiểm tra backup không rỗng, dump exit code, checksum, bản ngoài VPS và thông báo thất bại.
5. Mỗi tuần restore một bản thử; trước production chọn backup online/PITR phù hợp thay vì tự động dừng app hàng đêm theo lab.

## 15. Cập nhật, rollback và reboot

### 15.1. Lưu bản đang chạy trước khi build mới

Đầu tiên hoàn tất backup ở bước 14. Sau đó **VPS**:

```bash
cd /opt/nrapp/backend
NRAPP_RELEASE=$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p /opt/nrapp/releases
chmod 700 /opt/nrapp/releases
for service in gateway auth user mail chat todo workschedule canteen payment; do
  NRAPP_CONTAINER_ID=$(dc ps -q "$service")
  test -n "$NRAPP_CONTAINER_ID" || break
  NRAPP_IMAGE_ID=$(docker inspect --format '{{.Image}}' "$NRAPP_CONTAINER_ID")
  docker image tag "$NRAPP_IMAGE_ID" "nrapp-rollback/$service:$NRAPP_RELEASE" || break
done
docker image ls --filter 'reference=nrapp-rollback/*'
```

Xác nhận **đủ 9 image cùng tag thời gian** rồi mới tiếp tục. Tag rollback giữ image cũ ngay cả khi build mới dùng lại tag mặc định; nó chỉ tồn tại trên VPS này. Khi cần khôi phục sang VPS khác, lưu/push image vào nơi riêng được quản lý quyền.

Tạo override rollback từ tag vừa lưu:

```bash
printf 'services:\n' > "/opt/nrapp/releases/rollback-$NRAPP_RELEASE.yaml"
for service in gateway auth user mail chat todo workschedule canteen payment; do
  printf '  %s:\n    image: nrapp-rollback/%s:%s\n' "$service" "$service" "$NRAPP_RELEASE" \
    >> "/opt/nrapp/releases/rollback-$NRAPP_RELEASE.yaml"
done
```

Lưu cùng release: commit ID/source archive, `compose.vps.yaml`, env mã hóa, schema/migration version và danh sách image hạ tầng đang chạy. Không chỉ lưu source vì base image/dependency registry có thể thay đổi.

### 15.2. Cập nhật một phiên bản mà không build trên VPS

1. Lưu backup và đủ 9 image rollback ở 15.1.
2. Sửa/test source trên máy cá nhân. Xem thay đổi env, API/event contract và migration trước khi deploy.
3. Lặp bước 10.1 với tag mới **`vps-lab-002`** thay `vps-lab-001` ở lệnh build, tên archive/checksum và tên tag. Build đủ 9 image cho tag mới; Docker cache giúp service không đổi ít tốn công hơn.
4. Chuyển source/config cần thiết lên VPS, giữ env, cấu hình VPS và thư mục `logger/vps-minimal`. Import image/checksum thành công, kiểm tra đủ tag rồi đổi `NRAPP_IMAGE_TAG=vps-lab-002` trong root `.env`.
5. Kiểm tra Compose và cập nhật tuần tự, Gateway sau cùng:

```bash
cd /opt/nrapp/backend
dc --profile app config --quiet
(
  set -e
  for service in user mail auth chat todo workschedule canteen payment gateway; do
    dc --profile app up -d --no-deps --no-build --pull never --wait --wait-timeout 300 "$service"
  done
)
curl --fail --silent --show-error https://api.example.com/health
```

`--no-deps` ở đây dùng khi hạ tầng và dependency đã chạy từ release trước; nếu đang dựng mới dùng bước 10.4–10.5. Nếu một bước fail, ngừng deploy, xem log và rollback release; không chạy tiếp như đã thành công. Thử lại login/profile/socket/nghiệp vụ liên quan và xem RAM/queue sau deploy.

Một replica khi recreate có thể gián đoạn ngắn, Socket.IO reconnect; thay đổi API/event contract không tương thích cần maintenance window. Đây chưa phải quy trình zero-downtime.

Sau nghiệm thu, xóa đúng archive truyền tạm, giữ image rollback trước đó và bản archive ở máy cá nhân. Đổi env cần `up -d` để recreate; `docker restart` dùng env cũ. Nếu sửa thư viện observability dùng chung, vẫn rebuild đủ app dù đã tắt SDK trên VPS. Chỉ dùng build trực tiếp VPS theo phương án bảo trì ở 10.3 khi máy build ngoài không có.

### 15.3. Rollback image

Nếu lỗi do bản app mới và database/event contract vẫn tương thích, chọn **tag release cụ thể đã lưu**, rồi:

```bash
cd /opt/nrapp/backend
docker compose --env-file .env \
  -f compose.yaml -f compose.vps.yaml \
  -f /opt/nrapp/releases/rollback-REPLACE_TIMESTAMP.yaml \
  --profile app up -d --no-build --pull never --wait --wait-timeout 300
```

Kiểm tra image ID/tag đúng và thử nghiệp vụ lại. **Rollback image không rollback database.** Nếu migration mới không tương thích, chọn forward-fix hoặc phục hồi có kế hoạch, không chạy `migration:revert` mù quáng. Phục hồi PostgreSQL payment mà không xét MongoDB đơn hàng/RabbitMQ có thể lệch đối soát.

Sau rollback, ghi lại file rollback đang được dùng. Helper `dc` cơ bản không chứa file rollback; lần deploy tiếp phải chọn rõ cấu hình/image đích, tránh vô tình quay về image lỗi.

### 15.4. Thử tự phục hồi tiến trình và reboot VPS

Chỉ thử trên dữ liệu lab, không trong lúc chuyển tiền hoặc import dữ liệu:

1. Kiểm tra đủ 9 app có restart policy đúng:

```bash
for service in gateway auth user mail chat todo workschedule canteen payment; do
  NRAPP_CONTAINER_ID=$(dc ps -q "$service")
  docker inspect --format '{{.Name}} {{.HostConfig.RestartPolicy.Name}}' "$NRAPP_CONTAINER_ID"
done
```

2. Tạo một todo/message có ID đã ghi; xác nhận payment/đơn thử hiện có.
3. Bài dừng chủ động một service để xem ảnh hưởng và health/log; minimal chưa cấu hình cảnh báo ngoài:

```bash
dc --profile app stop chat
curl -i 'https://api.example.com/socket.io/?EIO=4&transport=polling'
dc --profile app up -d --no-build --pull never --wait --wait-timeout 300 chat
```

Chat đang stop không tự bật lại do `unless-stopped`; lệnh `up` khôi phục. Auth/Todo còn hoạt động nếu dependency không bị ảnh hưởng; Gateway `/health` có thể vẫn xanh dù Chat down.

4. Sau khi tất cả đã hoạt động lại, reboot:

```bash
sudo reboot
```

5. SSH lại khi máy lên, chạy:

```bash
systemctl is-active docker nginx
cd /opt/nrapp/backend
dc --profile app ps
mc ps
curl --fail --silent --show-error https://api.example.com/health
```

6. Thử login/OTP, xem dữ liệu cũ, socket reconnect và migration. Các app có thể retry dependency trong lúc reboot; Compose `depends_on` không đảm bảo thứ tự khởi động lại do Docker daemon theo mọi tình huống. Nếu không tự phục hồi, đọc log và bổ sung retry/lifecycle phù hợp trước khi coi đạt bài reboot.

### 15.5. Các lệnh không dùng tùy tiện

- Không `docker compose down -v`: `-v` xóa named volume thuộc project.
- Không `docker system prune --volumes`, không xóa `/var/lib/docker` để chữa lỗi app.
- Không đổi major PostgreSQL bằng cách thay tag và reuse volume mà chưa có kế hoạch nâng cấp.
- Không kỳ vọng đổi `POSTGRES_PASSWORD`/`RABBITMQ_DEFAULT_PASS` env sẽ tự đổi password trên volume đã khởi tạo; phải thay trong DB/broker rồi đồng bộ client.
- Không chạy `npm run dev` trên VPS này vì script có thể dừng app container để chạy watch.
- Không dùng `--remove-orphans` với wrapper logger; script hiện chủ động chặn option này.

## 16. Tra cứu lỗi thường gặp

| Triệu chứng | Chỗ kiểm tra trước | Cách xử lý theo dự án |
|---|---|---|
| `network nrapp-observability ... not found` | Network external đã tạo thủ công chưa? | Chạy bước 10.2; giữ tên network root/logger khớp nhau |
| Compose báo thiếu secret | `.env`, thư mục làm việc, `dc config --quiet` | Điền biến root bắt buộc; không chỉ điền env app |
| Mail/User kết nối sai broker hoặc dùng `guest` | Override uppercase RabbitMQ có được merge? | Dùng `dc` với file VPS; áp bộ biến chung cho 5 app như bước 9 |
| OTP API báo gửi nhưng không có email | Mail health/log, inbox spam, SMTP outbound, queue/DLQ | Sửa SMTP/consumer; không liên tục gọi login vượt cooldown |
| `MongoServerSelectionError` | Atlas allowlist, DNS SRV, password/URI, cluster trạng thái | Allow đúng IP outbound; URI `/nrapp`; kiểm tra từ container |
| `querySrv` timeout | DNS outbound, DNS trong container | Kiểm tra DNS nhà cung cấp; Chat còn gọi `dns.setServers(['8.8.8.8','8.8.4.4'])` trong bootstrap nên resolver này cũng phải truy cập được |
| Transaction chỉ chạy trên replica set | Deployment MongoDB | Dùng Atlas/replica set; không tắt transaction để né lỗi |
| JWT 401 / chữ ký internal sai | Secret hai đầu, env override, thời gian NTP | Đồng bộ JWT/Auth/internal secret đúng bảng; recreate container |
| Username trống dù login được | User profile/queue, log Auth đọc User | Kiểm tra `user-profile-sync`, record cùng ID và upstream User |
| Nginx 502 | `127.0.0.1:3000/health`, container Gateway, error log Nginx | Khôi phục Gateway hoặc đúng upstream port; chưa phải DNS lỗi |
| API được nhưng Chat realtime không được | `/socket.io`, headers Upgrade, JWT `auth.token` | Dùng đúng Nginx mẫu, đúng domain/path, token access phù hợp |
| Tất cả user cùng 429 | `request.ip`, trust proxy, X-Forwarded-For | Hoàn thành bước 9.4; không chỉ tăng rate limit lên cực lớn |
| Upload 413 | Nginx log và parser/route upload app | Xác định lớp giới hạn; tăng có chủ đích và test ảnh lớn |
| Casso 401 | Secret V2, header `x-casso-signature`, payload | Kiểm tra đúng cấu hình hai đầu; không bỏ verify |
| Payment paid nhưng đơn chưa đổi | Outbox, RabbitMQ, consumer Canteen và order ID | Đối chiếu cùng ID; kiểm tra retry/DLQ; smoke direct Payment có thể dùng order ID giả |
| `password authentication failed` PostgreSQL | Password thực trong volume so với env client | Password env khởi tạo không tự đổi DB cũ; đổi đúng ở server DB rồi đồng bộ |
| Không có trace/log Grafana hoặc p95 app | Phạm vi minimal đã tắt OTLP/Loki | Đây là tính năng đã bỏ; đọc `dc logs`, đo host bằng hai scrape target; không bật full stack để chữa `No data` |
| Không nhận cảnh báo ngoài | Minimal chưa cấu hình kênh thông báo | Kiểm tra thủ công theo 13; cảnh báo ngoài là bước thêm sau và phải test |
| Build exit 137 / container restart | OOM, heap và RAM/swap | Build ngoài VPS; kiểm tra từng limit/RSS theo 13.4; không tăng mọi limit cùng lúc |
| `no space left on device` | `df -h`, `docker system df`, log/metrics volumes | Dọn build cache/image không cần sau khi giữ rollback; không xóa volume dữ liệu |
| Registry lỗi IPv6/timeout | Kết nối registry từ host/Docker | Kiểm tra mạng IPv4/IPv6; 401 từ `/v2/` registry cho biết đã tới registry, không đồng nghĩa image lỗi |
| Container running nhưng unhealthy | Healthcheck log, dependency | Docker restart policy không tự sửa unhealthy; xử lý nguyên nhân |
| Gia hạn TLS fail | DNS A/AAAA, port 80, Nginx, Certbot scheduler | Chạy dry-run, sửa đường ACME; không chờ đến ngày hết hạn |

Nhóm lệnh chẩn đoán **VPS**:

```bash
cd /opt/nrapp/backend
dc --profile app ps
dc logs --tail=100 gateway auth user mail
dc logs --tail=100 payment canteen rabbitmq
mc logs --tail=100 prometheus grafana node-exporter
sudo tail -n 100 /var/log/nginx/error.log
sudo journalctl -u docker --since '30 minutes ago' --no-pager
sudo journalctl -k --since '30 minutes ago' --no-pager
free -h
df -h /
docker system df
docker stats --no-stream
```

Log có thể chứa dữ liệu request dù có sanitizer. Khi chia sẻ lỗi, bỏ email/token/URI/secret, giữ service, timestamp, request ID, error code và stack cần thiết.

## 17. Lịch thực hành và tiêu chí hoàn thành

### 17.1. Lộ trình 7 buổi

Thời lượng là ước lượng cho người mới, chưa tính thời gian xét duyệt tài khoản VPS/SMTP/domain.

| Buổi | Thời lượng dự kiến | Làm theo bước | Sản phẩm phải có |
|---|---|---|---|
| 1 — trước khi thuê | 1–2 giờ | 1–3 | Chốt gói/chi phí, hiểu sơ đồ, chuẩn bị key và tài khoản |
| 2 — Linux/VPS | 2–3 giờ | 4–5 | VPS SSH bằng deploy, firewall, Docker và Node hoạt động |
| 3 — cấu hình | 2–4 giờ | 6–9 | Source đầy đủ, Atlas/SMTP, env và override hợp lệ |
| 4 — chạy backend | 2–4 giờ | 10 | Image build trên máy cá nhân, import lên VPS; đủ 15 container, migration đúng |
| 5 — public API | 2–3 giờ | 11–12 | HTTPS, OTP/profile, Chat/Todo/lịch/Canteen và payment lab |
| 6 — vận hành | 2–4 giờ + theo dõi 24 giờ | 13–14 | Đạt ngân sách RAM 4 GB, dashboard host, backup ngoài VPS và restore thử |
| 7 — diễn tập | 2–3 giờ | 15–16 | Deploy một thay đổi, rollback, reboot và ghi báo cáo |

Nếu mắc lỗi, giữ lại thông tin lỗi và checkpoint đã đạt. Ví dụ chưa gửi được SMTP thì giải quyết trước buổi kiểm tra OTP, không nhảy qua rồi coi login hoàn thành.

### 17.2. Checklist nghiệm thu đầy đủ

**Truy cập và hạ tầng**

- [ ] Có deploy SSH bằng key, sudo và console cứu hộ.
- [ ] Chỉ cửa vào public dự kiến hoạt động: SSH theo rule, HTTP/HTTPS.
- [ ] Từ Internet không kết nối được Gateway 3000, app 4000/5000–5006, DB/cache/broker/monitoring.
- [ ] Đủ **9 app + 3 hạ tầng + 3 giám sát = 15 container**, không có full logger/pgAdmin chạy thừa.
- [ ] Image build ngoài VPS; cấu hình test RAM 4 GB đạt tiêu chí mục 13.4.
- [ ] `OTEL_SDK_DISABLED=true` trên đủ 9 app; log JSON vẫn đọc được.
- [ ] App restart policy là `unless-stopped`; reboot đã thử thực tế.
- [ ] MongoDB lab tách khỏi dữ liệu thật và hỗ trợ transaction.
- [ ] Payment có migration, không bật `synchronize:true`.

**Ứng dụng**

- [ ] Domain HTTPS hợp lệ, HTTP redirect, renewal dry-run thành công.
- [ ] Register → profile sync → login → OTP mail → verify → me → refresh hoạt động.
- [ ] User thường không thực hiện được thao tác admin.
- [ ] Hai thiết bị gửi/nhận Chat realtime; upload Cloudinary được.
- [ ] Todo CRUD được; lịch duyệt và xuất kho transaction thành công.
- [ ] Canteen tạo/xử lý đơn với dữ liệu mẫu đúng.
- [ ] Payment tạo QR từ giá server, webhook sai bị từ chối, replay không nhân đôi.
- [ ] Đã phân biệt payment smoke giả lập với webhook/giao dịch thật.
- [ ] Client IP qua Nginx không bị gom sai vào một bucket chung cho mọi mạng.

**Vận hành**

- [ ] Minimal smoke ở 13.1 thành công: 3 monitoring container, 2 targets up; biết giới hạn liveness Auth/Gateway.
- [ ] Dashboard CPU/RAM/disk có dữ liệu; log xem bằng `dc logs`.
- [ ] Hiểu các chức năng đã bỏ: trace/OTLP, log tập trung, metrics app và cảnh báo ngoài chưa cấu hình.
- [ ] Có backup PostgreSQL/Mongo/config, bản ngoài VPS, checksum.
- [ ] Đã restore vào môi trường riêng và kiểm tra dữ liệu bằng ứng dụng.
- [ ] Có image/source release trước, đã thử rollback tương thích schema.
- [ ] Ghi RAM/CPU/disk, RSS/limit, swap và độ dài queue; không OOM/restart lặp sau bài chạy thử.

### 17.3. Mẫu ghi chép sau mỗi buổi

```markdown
Ngày/giờ:
VPS/gói/region:
Source commit hoặc tên bản source:
Image release/tag:
Bước đã làm:
Kết quả mong đợi:
Kết quả thực tế:
Lỗi + service + request ID (không ghi secret):
Cách đã xử lý:
RAM/disk trước và sau:
Backup gần nhất và nơi lưu ngoài VPS:
Bước tiếp theo:
```

## 18. Khi nào nâng cấp kiến trúc

### 18.1. Tối ưu sau khi có số đo

1. Đo RSS từng container, RAM host, swap, CPU và queue ở trạng thái idle/tải. p95/5xx/throughput dùng kết quả công cụ thử tải hoặc bổ sung instrumentation riêng; minimal dashboard hiện không có các metric API này.
2. Chọn API đọc ít tác động để thử tải tăng dần từ thấp; không dùng login OTP, upload ảnh hàng loạt hoặc payment webhook làm bài tải đầu tiên.
3. Dừng tăng tải khi OOM, 5xx hoặc p95 vượt mục tiêu bạn chọn; xác định app/DB/queue/network là nơi nghẽn.
4. Đặt memory/CPU limit theo số đo, điều chỉnh connection pool và retention; build trên máy/CI khác nếu build làm gián đoạn VPS.
5. Đừng suy từ 15 container ra số user hỗ trợ; cần dữ liệu workload và số đo thực.

### 18.2. Trước khi dùng cho dữ liệu thật

| Hạng mục | Việc còn phải hoàn thiện |
|---|---|
| Khả năng chịu lỗi | Một VPS là một điểm lỗi; quyết định mức gián đoạn chấp nhận và có môi trường phục hồi |
| Backup | Backup online/PITR phù hợp, restore định kỳ và đối soát dữ liệu liên dịch vụ |
| Database | Cân nhắc managed PostgreSQL/Atlas tier đáp ứng workload và backup; nâng version có kế hoạch |
| Traces | Jaeger persistent storage/retention phù hợp nếu cần điều tra qua reboot |
| Entry/API | CORS theo origin, xác minh Google token, test IP/rate limiting theo topology thực |
| Secret | Quy trình rotate từng loại, giữ tương thích hai đầu; không đổi toàn bộ secret mỗi deploy |
| Delivery | CI build/test, image registry riêng, pin digest/release, deployment có kiểm tra và rollback |
| Tài chính | Đối soát transaction/webhook/outbox/đơn hàng, không chỉ dựa dashboard UI |

### 18.3. Tách hoặc scale service có lý do

- Giám sát chiếm nhiều RAM/disk: chuyển observability sang máy/dịch vụ riêng sau khi đo; giữ private network/firewall.
- PostgreSQL I/O/backup trở thành điểm nghẽn: tách DB hoặc dùng managed DB; xem lại TLS vì source hiện `PAYMENT_DB_SSL=true` dùng `rejectUnauthorized:false`, cần cấu hình CA/xác minh certificate trước kết nối DB ngoài máy cho môi trường thật.
- Gateway cần nhiều replica: rate limiter hiện dùng `Map` trong RAM; cần bộ đếm chung hoặc rate limiting tại ingress.
- Chat cần nhiều replica: `userSocketMap` hiện nằm trong RAM một process; cần Socket.IO adapter/chia sẻ presence và routing phù hợp, kiểm tra sticky session nếu dùng polling.
- Tách máy không tự tạo high availability nếu vẫn chỉ có một DB hoặc một broker. Tăng độ phức tạp sau khi đã thành thạo deploy, backup, restore và đo tải trên một VPS.

## Tài liệu đối chiếu

### Trong dự án

- [Backend rules](../backend/CLAUDE.md), [Backend README](../backend/README.md).
- [Docker Compose Guide](DOCKER_COMPOSE_GUIDE.md), [Docker và PostgreSQL](HUONG_DAN_DOCKER_VA_POSTGRES.md).
- [Hướng dẫn observability](HUONG_DAN_LUONG_OBSERVABILITY_VA_CAU_HINH.md), [Logger README](../backend/logger/README.md).
- [Payment Guide](PAYMENT_SERVICE_GUIDE.md), [pgAdmin cho team](HUONG_DAN_PGADMIN_TEAM.md).

### Nguồn chính thức bên ngoài

- [Docker Engine trên Ubuntu](https://docs.docker.com/engine/install/ubuntu/) và [firewall](https://docs.docker.com/engine/network/packet-filtering-firewalls/).
- [DigitalOcean Droplet pricing](https://www.digitalocean.com/pricing/droplets), [Hetzner server overview](https://docs.hetzner.com/cloud/servers/overview/).
- [NVM](https://github.com/nvm-sh/nvm), [Express behind proxies](https://expressjs.com/en/guide/behind-proxies/).
- [Nginx WebSocket proxy](https://nginx.org/en/docs/http/websocket.html), [Certbot Nginx](https://certbot.eff.org/instructions?ws=nginx&os=snap).
- [Atlas Free limitations](https://www.mongodb.com/docs/atlas/reference/free-shared-limitations/), [MongoDB transactions](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/), [mongodump](https://www.mongodb.com/docs/database-tools/mongodump/).

Các thông số chọn VPS, thời gian thực hành, retention lab và mục tiêu RPO/RTO là đề xuất cho dự án này. Giá, giao diện nhà cung cấp và phiên bản phần mềm cần kiểm tra lại khi thực hiện; cấu hình/source trong repo là căn cứ cho tên service, port, route và biến môi trường nêu trong tài liệu.
