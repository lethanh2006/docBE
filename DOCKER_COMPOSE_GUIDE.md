# Chạy toàn bộ NRApp Backend bằng Docker Compose trên Linux Mint

Tài liệu này áp dụng cho thư mục `backend` hiện tại và chạy một lần toàn bộ:

- 10 ứng dụng: Gateway, Auth, User, Mail, Chat, Todo, Workschedule, Canteen, Payment và Logger.
- 2 dịch vụ hạ tầng: Redis và RabbitMQ Management.
- MongoDB, Cloudinary và SMTP tiếp tục dùng dịch vụ bên ngoài được khai báo trong các file `.env` hiện có.

Lệnh khởi động chính sau khi hoàn tất thiết lập:

```bash
cd /media/thanhle/D/pj1/backend
docker compose up -d --build --wait --wait-timeout 300
```

## 1. Kiến trúc sau khi Docker hóa

```text
Mobile/Web
    |
    v
Gateway :3000
    |
    +--> Auth :4000 --------> Redis + RabbitMQ + MongoDB
    +--> User :5000 --------> Redis + RabbitMQ + MongoDB
    +--> Chat :5002 --------> User + MongoDB + Cloudinary
    +--> Todo :5003 --------> User + MongoDB
    +--> Workschedule :5004 -> User + MongoDB
    +--> Canteen :5005 -----> User + Redis + RabbitMQ + MongoDB
    +--> Payment :5006 -----> MongoDB

Mail :5001 <--------------- RabbitMQ + SMTP
Logger :5007 --------------> named volume logger_data
```

Trong mạng Docker, các container gọi nhau bằng tên service như `redis`, `rabbitmq`, `user` và `auth`. Không dùng `localhost` để gọi container khác.

## 2. Các file đã được chuẩn bị

- `compose.yaml`: định nghĩa đầy đủ 12 container.
- `.env.example`: mẫu cấu hình Compose, port và RabbitMQ credential.
- `.dockerignore`: ngăn `.env`, `node_modules`, `.git` và log bị đưa vào image.
- `docker/node-service.Dockerfile`: Dockerfile multi-stage dùng chung cho các service TypeScript.
- `docker/logger.Dockerfile`: Dockerfile riêng cho Logger JavaScript.
- Mỗi ứng dụng có endpoint `/health` để Compose kiểm tra trạng thái.

Các image hạ tầng đang được pin để tránh tự nâng major version ngoài ý muốn:

- Redis: `redis:7.4.10-alpine`
- RabbitMQ: `rabbitmq:4.2.7-management-alpine`
- Node.js build/runtime: `node:22-alpine`

## 3. Cài Docker Engine và Docker Compose trên Linux Mint

Máy hiện tại đã nhận được `Docker 29.7.1` và `Docker Compose v5.4.0`. Nếu hai lệnh sau chạy được thì bỏ qua phần cài mới và chuyển đến mục 4:

```bash
docker --version
docker compose version
```

Linux Mint là bản phân phối dựa trên Ubuntu. Docker ghi rõ các bản Ubuntu derivative không được hỗ trợ chính thức, dù repository Ubuntu thường hoạt động. Xem hướng dẫn mới nhất trước khi cài:

- <https://docs.docker.com/engine/install/ubuntu/>
- <https://docs.docker.com/engine/install/linux-postinstall/>

### 3.1. Kiểm tra Ubuntu base codename của Linux Mint

```bash
. /etc/os-release
printf 'Mint codename: %s\nUbuntu base codename: %s\n' "$VERSION_CODENAME" "$UBUNTU_CODENAME"
```

Repository Docker phải sử dụng `UBUNTU_CODENAME`, ví dụ `jammy` hoặc `noble`, không dùng Mint codename.

### 3.2. Gỡ package có thể xung đột

Kiểm tra trước:

```bash
dpkg -l docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc 2>/dev/null
```

Chỉ khi đang cài mới Docker Engine chính thức, chạy:

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc
```

Lệnh này không tự xóa dữ liệu trong `/var/lib/docker`, nhưng vẫn nên đọc danh sách package mà `apt` chuẩn bị gỡ trước khi xác nhận.

### 3.3. Thêm repository Docker chính thức

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Tạo repository source:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Cài Docker Engine và Compose plugin:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker.service containerd.service
```

### 3.4. Cho phép user hiện tại dùng Docker

Group `docker` có quyền tương đương root. Không thêm user không tin cậy vào group này.

```bash
getent group docker || sudo groupadd docker
sudo usermod -aG docker "$USER"
```

Đăng xuất rồi đăng nhập lại. Có thể áp dụng tạm cho terminal hiện tại bằng:

```bash
newgrp docker
```

Kiểm tra:

```bash
id
docker info
docker run --rm hello-world
docker compose version
```

Không sửa socket bằng `sudo chmod 666 /var/run/docker.sock`; cách đó mở quyền Docker cho mọi user trên máy.

## 4. Chuẩn bị cấu hình project

### 4.1. Đi tới thư mục Backend

```bash
cd /media/thanhle/D/pj1/backend
pwd
```

Kết quả phải là:

```text
/media/thanhle/D/pj1/backend
```

### 4.2. Tạo `.env` riêng cho Compose

```bash
cp .env.example .env
openssl rand -hex 32
nano .env
```

Dán chuỗi từ `openssl` vào `RABBITMQ_PASSWORD`. Không dùng `guest/guest` và không commit file `.env`.

Ví dụ cấu trúc, không dùng nguyên mật khẩu mẫu:

```dotenv
COMPOSE_PROJECT_NAME=nrapp-backend
RABBITMQ_USER=nrapp_dev
RABBITMQ_PASSWORD=<chuỗi-hex-vừa-tạo>
GATEWAY_BIND_IP=0.0.0.0
```

Các biến port còn lại có sẵn trong `.env.example`. Giữ mặc định nếu máy không trùng port.

### 4.3. Kiểm tra file `.env` của từng service

Các file phải tồn tại:

```bash
for service in auth user mail chat todo workschedule canteen payment gateway; do
  test -f "$service/.env" && printf 'OK  %s/.env\n' "$service" || printf 'THIEU  %s/.env\n' "$service"
done
```

Kiểm tra tên biến mà không in giá trị secret:

```bash
for file in auth/.env user/.env mail/.env chat/.env todo/.env workschedule/.env canteen/.env payment/.env gateway/.env; do
  printf '\n[%s]\n' "$file"
  sed -E '/^[[:space:]]*($|#)/d; s/^([^=]+)=.*/\1=<set>/' "$file"
done
```

Tối thiểu cần kiểm tra:

| Service | Biến quan trọng |
|---|---|
| Auth | `MONGO_URL`, `JWT_SECRET` |
| User | `MONGO_URL`, `JWT_SECRET` |
| Mail | `SMTP_USER`, `SMTP_PASS` |
| Chat | `MONGO_URL`, `JWT_SECRET`, các biến `CLOUDINARY_*` |
| Todo | `MONGO_URL`, `JWT_SECRET` |
| Workschedule | `MONGO_URL`, `JWT_SECRET` |
| Canteen | `MONGO_URL`, `JWT_SECRET` |
| Payment | `MONGO_URL`, `JWT_SECRET` |
| Gateway | `JWT_SECRET` |

`JWT_SECRET` của Gateway phải phù hợp với token mà Auth phát hành. Compose sẽ tự ghi đè URL nội bộ và RabbitMQ credential, vì vậy không cần sửa `localhost` trong `.env` của từng service khi chạy full stack.

## 5. Kiểm tra Compose trước khi chạy

Kiểm tra cú pháp mà không in toàn bộ secret:

```bash
docker compose config --quiet
```

Liệt kê service:

```bash
docker compose config --services
```

Phải có 12 tên:

```text
redis
rabbitmq
auth
user
mail
chat
todo
workschedule
canteen
payment
logger
gateway
```

Không nên chạy `docker compose config` rồi gửi toàn bộ output lên nơi công cộng vì output đã render có thể chứa secret từ các file `.env`.

Kiểm tra port đang bị chiếm:

```bash
ss -ltnp | grep -E ':(3000|4000|5000|5001|5002|5003|5004|5005|5006|5007|5672|6379|15672)\b' || true
```

Nếu có xung đột, đổi biến `*_HOST_PORT` tương ứng trong `backend/.env`. Không cần đổi port nội bộ của container.

## 6. Build và khởi động toàn bộ hệ thống

### 6.1. Kéo Redis và RabbitMQ

```bash
docker compose pull redis rabbitmq
```

### 6.2. Build 10 ứng dụng

```bash
docker compose build
```

Build sạch hoàn toàn chỉ dùng khi nghi cache bị lỗi:

```bash
docker compose build --no-cache
```

### 6.3. Chạy và đợi healthcheck

```bash
docker compose up -d --wait --wait-timeout 300
```

Hoặc build và chạy bằng một lệnh:

```bash
docker compose up -d --build --wait --wait-timeout 300
```

Lần đầu phải tải image và cài package nên có thể mất vài phút. Các lần sau sẽ nhanh hơn nhờ cache.

### 6.4. Kiểm tra trạng thái

```bash
docker compose ps
```

Các service ứng dụng, Redis và RabbitMQ phải ở trạng thái `running` hoặc `healthy`.

Xem tài nguyên đang dùng:

```bash
docker stats --no-stream
```

Docker Compose giảm số terminal và chuẩn hóa môi trường; nó không làm 10 tiến trình Node tự nhiên dùng ít RAM hơn. Chỉ chạy những service cần thiết nếu máy yếu.

## 7. Địa chỉ truy cập

| Thành phần | Địa chỉ từ máy host |
|---|---|
| Gateway | `http://localhost:3000` |
| Gateway Swagger | `http://localhost:3000/api-docs` |
| Auth | `http://localhost:4000` |
| User | `http://localhost:5000` |
| Mail | `http://localhost:5001` |
| Chat | `http://localhost:5002` |
| Todo | `http://localhost:5003` |
| Workschedule | `http://localhost:5004` |
| Canteen | `http://localhost:5005` |
| Payment | `http://localhost:5006` |
| Logger | `http://localhost:5007` |
| Redis | `127.0.0.1:6379` |
| RabbitMQ AMQP | `127.0.0.1:5672` |
| RabbitMQ Dashboard | `http://localhost:15672` |

Đăng nhập RabbitMQ Dashboard bằng `RABBITMQ_USER` và `RABBITMQ_PASSWORD` trong `backend/.env`.

Gateway mặc định bind `0.0.0.0:3000` để điện thoại thật trong LAN có thể truy cập. Các service còn lại chỉ bind `127.0.0.1`, không phơi trực tiếp ra LAN.

Lấy IP LAN của máy:

```bash
hostname -I
```

Từ điện thoại dùng `http://<IP-LAN-CUA-MAY>:3000`, không dùng `localhost:3000`.

## 8. Kiểm tra health và hạ tầng

### 8.1. Kiểm tra tất cả HTTP service

```bash
for port in 3000 4000 5000 5001 5002 5003 5004 5005 5006 5007; do
  printf 'Port %s: ' "$port"
  curl -fsS "http://127.0.0.1:${port}/health" && printf '\n' || printf 'FAILED\n'
done
```

### 8.2. Kiểm tra Redis

```bash
docker compose exec redis redis-cli ping
```

Kết quả đúng:

```text
PONG
```

### 8.3. Kiểm tra RabbitMQ

```bash
docker compose exec rabbitmq rabbitmq-diagnostics -q ping
docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers
```

Sau khi Auth/User/Mail chạy, danh sách queue ít nhất có thể xuất hiện `send-otp` và `user-profile-sync`.

## 9. Các lệnh vận hành hàng ngày

Khởi động hoặc đồng bộ lại cấu hình:

```bash
cd /media/thanhle/D/pj1/backend
docker compose up -d
```

Dừng nhưng giữ container:

```bash
docker compose stop
```

Khởi động lại container đã dừng:

```bash
docker compose start
```

Restart một service:

```bash
docker compose restart auth
```

Xem 100 dòng log cuối:

```bash
docker compose logs --tail 100
```

Theo dõi log toàn hệ thống:

```bash
docker compose logs -f --tail 100
```

Theo dõi một vài service:

```bash
docker compose logs -f --tail 100 gateway auth user rabbitmq
```

Dừng và xóa container/network nhưng giữ named volume:

```bash
docker compose down
```

`restart: unless-stopped` giúp container tự chạy lại sau khi Docker daemon hoặc máy khởi động lại, trừ khi bạn đã chủ động stop chúng.

## 10. Quy trình sau khi sửa code

Build lại một service và khởi động phần phụ thuộc cần thiết:

```bash
docker compose up -d --build auth
```

Build lại nhiều service:

```bash
docker compose up -d --build gateway auth user
```

Build lại toàn bộ:

```bash
docker compose up -d --build --wait --wait-timeout 300
```

Dockerfile hiện tại tạo image kiểu production, không bind-mount source và không bật hot reload. Vì vậy thay đổi code cần build lại image.

## 11. Chỉ chạy một phần hệ thống để tiết kiệm RAM

Chỉ chạy hạ tầng:

```bash
docker compose up -d redis rabbitmq
```

Chạy luồng Auth/User/Mail:

```bash
docker compose up -d redis rabbitmq auth user mail gateway
```

Chạy Gateway với Todo và User:

```bash
docker compose up -d redis rabbitmq auth user todo gateway
```

Lưu ý `depends_on` có thể tự khởi động thêm dependency. Kiểm tra kết quả bằng:

```bash
docker compose ps
```

Nếu chạy Node service trực tiếp ngoài Docker trong khi chỉ Redis/RabbitMQ nằm trong Docker, phải đồng bộ `Rabbitmq_Username` và `Rabbitmq_Password` trong `.env` của Auth, User, Canteen và Mail với `backend/.env`. Host lúc đó tiếp tục dùng `Rabbitmq_Host=localhost` và `REDIS_URL=redis://127.0.0.1:6379`.

## 12. Backup và reset dữ liệu local

Tạo thư mục backup:

```bash
mkdir -p backups
```

Yêu cầu Redis ghi snapshot rồi sao chép ra host:

```bash
docker compose exec redis redis-cli BGSAVE
docker compose cp redis:/data/dump.rdb ./backups/redis-dump.rdb
```

Export RabbitMQ definitions:

```bash
docker compose exec rabbitmq rabbitmqctl export_definitions /tmp/rabbitmq-definitions.json
docker compose cp rabbitmq:/tmp/rabbitmq-definitions.json ./backups/rabbitmq-definitions.json
```

Xóa container nhưng giữ dữ liệu:

```bash
docker compose down
```

Xóa cả Redis/RabbitMQ/Logger data để làm lại từ đầu:

```bash
docker compose down -v
```

`down -v` xóa named volume và dữ liệu local không thể khôi phục nếu chưa backup. Đọc kỹ danh sách volume trước khi xác nhận:

```bash
docker compose volumes
```

RabbitMQ chỉ dùng `RABBITMQ_DEFAULT_USER` và `RABBITMQ_DEFAULT_PASS` khi volume được khởi tạo lần đầu. Nếu đổi credential trong `.env` nhưng giữ volume cũ, credential cũ vẫn tồn tại. Với dữ liệu dev không cần giữ, có thể backup rồi `down -v` và `up` lại.

## 13. Xử lý lỗi thường gặp

### 13.1. `permission denied while trying to connect to the Docker daemon socket`

```bash
id
getent group docker
ls -l /var/run/docker.sock
```

Nếu user chưa thuộc group:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker info
```

Nếu vẫn lỗi, đăng xuất rồi đăng nhập lại hoặc reboot.

### 13.2. Service `unhealthy`

```bash
docker compose ps
docker compose logs --tail 200 <ten-service>
docker inspect --format '{{json .State.Health}}' "$(docker compose ps -q <ten-service>)"
```

Ví dụ:

```bash
docker compose logs --tail 200 auth
docker inspect --format '{{json .State.Health}}' "$(docker compose ps -q auth)"
```

### 13.3. MongoDB không kết nối được

Kiểm tra log:

```bash
docker compose logs --tail 200 auth user chat todo workschedule canteen payment
```

Kiểm tra:

- `MONGO_URL` có đúng không.
- MongoDB Atlas có cho phép IP public hiện tại không.
- DNS và Internet của host có hoạt động không.
- Password có ký tự đặc biệt đã URL-encode đúng chưa.

Không in hoặc gửi `MONGO_URL` đầy đủ vì chuỗi thường chứa username/password.

### 13.4. RabbitMQ báo authentication failed

```bash
docker compose logs --tail 200 rabbitmq auth user mail canteen
docker compose exec rabbitmq rabbitmqctl list_users
```

Nếu vừa đổi credential nhưng volume đã tồn tại, đọc lại lưu ý tại mục 12.

### 13.5. Port đã được sử dụng

```bash
ss -ltnp | grep ':5000\b'
```

Đổi port host trong `backend/.env`, ví dụ:

```dotenv
USER_HOST_PORT=5100
```

Sau đó áp dụng:

```bash
docker compose up -d --force-recreate user
```

URL nội bộ của Gateway vẫn là `http://user:5000`; không đổi nó thành `5100`.

### 13.6. Mail không gửi OTP

```bash
docker compose logs -f --tail 200 mail rabbitmq auth
docker compose exec rabbitmq rabbitmqctl list_queues name messages consumers
```

Kiểm tra `SMTP_USER`, `SMTP_PASS`, quyền đăng nhập SMTP và queue `send-otp`. Nếu dùng Gmail, thường phải dùng App Password thay vì password tài khoản chính.

### 13.7. Build lỗi package hoặc cache

Thử build đúng service trước:

```bash
docker compose build --no-cache <ten-service>
```

Không xóa `package-lock.json`; image dùng `npm ci` để build có thể tái lập.

## 14. Cập nhật image an toàn

Xem image đang dùng:

```bash
docker compose images
```

Kéo đúng tag đã pin:

```bash
docker compose pull redis rabbitmq
docker compose up -d --wait --wait-timeout 300
```

Không tự đổi major Redis/RabbitMQ/Node rồi chạy thẳng. Nên đọc release notes, backup volume, build và test các luồng Auth/User/Mail/Canteen trước.

## 15. Giới hạn hiện tại của source code

- Payment container chạy ở port `5006`, nhưng Gateway source hiện chưa import module định tuyến Payment. `PAYMENT_SERVICE_URL` đã được truyền sẵn để dùng khi module đó được bổ sung.
- Logger chạy độc lập ở `5007`; hiện chưa thấy các service khác gửi log tới Logger qua URL chung.
- Healthcheck hiện xác nhận tiến trình HTTP đã phục vụ. Nó không kiểm tra sâu mọi truy vấn MongoDB, Redis hay RabbitMQ sau khi service đã chạy.
- Chat hiện có peer-dependency cũ giữa `multer-storage-cloudinary@4` và `cloudinary@2`. Docker build chỉ bật `--legacy-peer-deps` cho riêng Chat để tái lập đúng lockfile hiện tại; nên nâng hoặc thay package storage này trong một đợt refactor riêng.
- Đây là cấu hình development local. Khi production cần secret manager, TLS, reverse proxy, network policy, resource limits, backup/monitoring và không publish trực tiếp các port nội bộ.

## 16. Checklist hoàn tất

```text
[ ] docker info chạy không cần sudo
[ ] backend/.env đã tạo và đã đổi RABBITMQ_PASSWORD
[ ] .env của từng service có đủ secret cần thiết
[ ] docker compose config --quiet thành công
[ ] docker compose up -d --build --wait thành công
[ ] docker compose ps báo healthy
[ ] Redis trả PONG
[ ] RabbitMQ Dashboard đăng nhập được
[ ] 10 endpoint /health trả status ok
[ ] Gateway Swagger mở được
[ ] Điện thoại truy cập được IP-LAN:3000 nếu cần
```
