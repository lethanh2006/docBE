# Hướng dẫn đổi mật khẩu PostgreSQL và giới hạn container tự chạy

Tất cả lệnh trong tài liệu này được chạy từ thư mục backend:

```bash
cd /media/thanhle/D1/pj1/backend
```

Ba service hạ tầng cần được phép tự chạy khi mở máy là:

- `redis`
- `rabbitmq`
- `payment-postgres` (PostgreSQL/SQL)

Các lệnh bên dưới không xóa volume hay dữ liệu. Tuyệt đối không thêm tùy chọn `-v`.

## 1. Đổi mật khẩu PostgreSQL

PostgreSQL hiện dùng user `nrapp_payment`, database `nrapp_payment`, và volume lưu dữ liệu lâu dài. Vì vậy, chỉ sửa `PAYMENT_POSTGRES_PASSWORD` trong `.env` sẽ **không** đổi mật khẩu thật của user đã tồn tại trong PostgreSQL.

### Bước 1: Bảo đảm PostgreSQL đang chạy

```bash
npm run infra:up
docker compose ps payment-postgres
```

Chờ trạng thái của `payment-postgres` thành `healthy`.

### Bước 2: Dừng Payment trước khi đổi mật khẩu

```bash
docker compose stop payment
```

### Bước 3: Tạo mật khẩu mới

Nên dùng chuỗi hex dài để tránh lỗi với ký tự `$`, `#`, dấu cách hoặc dấu nháy trong file `.env`:

```bash
openssl rand -hex 32
```

Sao chép kết quả và lưu trong password manager. Không dán mật khẩu vào câu lệnh SQL trên command line vì nó có thể bị lưu vào shell history.

Mật khẩu tạo bởi lệnh vẫn xuất hiện trên màn hình/terminal scrollback. Không chụp hoặc chia sẻ terminal này, và xóa scrollback sau khi hoàn tất nếu máy có người khác sử dụng. Trước khi đổi, cũng nên giữ mật khẩu cũ trong password manager để có thể rollback; không sao lưu `.env` thành file plaintext như `.env.bak`.

### Bước 4: Đổi mật khẩu thật bên trong PostgreSQL

Mở `psql`:

```bash
docker compose exec payment-postgres \
  psql -U nrapp_payment -d nrapp_payment
```

Tại dấu nhắc `nrapp_payment=#`, nhập:

```text
\password nrapp_payment
```

Nhập mật khẩu mới hai lần. Khi thành công, thoát bằng:

```text
\q
```

Lệnh `\password` hỏi mật khẩu ở chế độ ẩn, an toàn hơn việc viết `ALTER ROLE ... PASSWORD '...'` trực tiếp trong terminal.

### Bước 5: Đồng bộ mật khẩu trong hai file cấu hình

Giới hạn quyền đọc file secret trước khi mở chúng:

```bash
chmod 600 .env payment/.env
```

Mở file cấu hình gốc:

```bash
nano .env
```

Tìm và thay giá trị sau, không nhập dấu `<` và `>`:

```dotenv
PAYMENT_POSTGRES_PASSWORD=<MAT_KHAU_MOI>
```

Sau đó mở cấu hình dùng khi chạy riêng service Payment trên máy:

```bash
nano payment/.env
```

Tìm và thay:

```dotenv
PAYMENT_DB_PASSWORD=<MAT_KHAU_MOI>
```

Không ghi mật khẩu thật vào `.env.example`.

Kiểm tra hai file đang chứa cùng một giá trị mà không in mật khẩu ra màn hình:

```bash
root_db_password="$(sed -n 's/^PAYMENT_POSTGRES_PASSWORD=//p' .env)"
payment_db_password="$(sed -n 's/^PAYMENT_DB_PASSWORD=//p' payment/.env)"

if [ "$root_db_password" = "$payment_db_password" ]; then
  echo 'OK: hai file dùng cùng mật khẩu'
else
  echo 'LỖI: mật khẩu trong hai file không khớp'
fi

unset root_db_password payment_db_password
```

### Bước 6: Tạo lại container PostgreSQL để biến môi trường khớp

```bash
docker compose up -d --no-deps --force-recreate payment-postgres
```

Lệnh này chỉ tạo lại container; named volume `payment_postgres_data` vẫn được giữ nên dữ liệu không bị xóa.
PostgreSQL sẽ gián đoạn trong thời gian ngắn khi container được tạo lại.

### Bước 7: Kiểm tra đúng bằng mật khẩu mới

Healthcheck `pg_isready` không chứng minh mật khẩu đúng. Hãy kiểm tra kết nối TCP có xác thực:

```bash
docker compose exec payment-postgres \
  psql -h 127.0.0.1 \
  -U nrapp_payment \
  -d nrapp_payment \
  -W \
  -c 'SELECT current_user, current_database();'
```

Nhập mật khẩu mới khi được hỏi. Kết quả cần chứa:

```text
nrapp_payment | nrapp_payment
```

Nếu hiện tại chỉ muốn chạy hạ tầng thì giữ `payment` ở trạng thái dừng. Khi cần chạy Payment lại, dùng Compose để container nhận cấu hình mới:

```bash
docker compose up -d --no-deps --force-recreate --wait payment
```

Không nên dùng `docker start nrapp-backend-payment-1` sau khi đổi mật khẩu, vì container cũ có thể vẫn giữ biến môi trường cũ.

### Nếu cần rollback mật khẩu

Mở PostgreSQL qua Unix socket:

```bash
docker compose exec payment-postgres \
  psql -U nrapp_payment -d nrapp_payment
```

Trong `psql`, chạy lại lệnh sau, nhập mật khẩu cũ đã lưu trong password manager, rồi thoát:

```text
\password nrapp_payment
\q
```

Khôi phục mật khẩu cũ tại `PAYMENT_POSTGRES_PASSWORD` trong `.env` và `PAYMENT_DB_PASSWORD` trong `payment/.env`, sau đó đồng bộ metadata container:

```bash
docker compose up -d --no-deps --force-recreate payment-postgres
```

Cuối cùng chạy lại lệnh kiểm tra TCP ở Bước 7 bằng mật khẩu cũ.

## 2. Xử lý ngay các container hiện có

Phần này dừng app nhưng không xóa container hoặc dữ liệu.

### Bước 1: Tắt tự khởi động và dừng các app backend

```bash
docker compose ps -a -q \
  auth user mail chat todo workschedule canteen payment gateway \
  | xargs -r docker update --restart=no

docker compose stop \
  auth user mail chat todo workschedule canteen payment gateway
```

### Bước 2: Tắt tự khởi động và dừng Loki, Alloy, Grafana

Ba container quan sát này thuộc project Compose riêng `nrapp-observability`:

```bash
docker ps -a -q \
  --filter 'label=com.docker.compose.project=nrapp-observability' \
  | xargs -r docker update --restart=no

docker ps -q \
  --filter 'label=com.docker.compose.project=nrapp-observability' \
  | xargs -r docker stop
```

### Bước 3: Cho phép đúng ba container hạ tầng tự chạy

```bash
docker compose ps -a -q redis rabbitmq payment-postgres \
  | xargs -r docker update --restart=unless-stopped

docker compose up -d --wait redis rabbitmq payment-postgres
```

Lệnh ngắn có sẵn để khởi động ba service này vào những lần sau:

```bash
npm run infra:up
```

### Bước 4: Kiểm tra restart policy và trạng thái

```bash
docker ps -a -q | xargs -r docker inspect \
  --format '{{.Name}} -> restart={{.HostConfig.RestartPolicy.Name}}; status={{.State.Status}}'
```

Kết quả mong muốn:

- Redis, RabbitMQ, PostgreSQL: `restart=unless-stopped`, `status=running`.
- Các app container đã tồn tại cùng Loki, Alloy, Grafana: `restart=no`, `status=exited`.

Sau lần reboot tự nhiên tiếp theo, kiểm tra nhanh:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Không cần chạy `sudo systemctl restart docker` chỉ để thử, vì thao tác đó ảnh hưởng tất cả container Docker trên máy.

## 3. Sửa Compose để thiết lập không bị quay lại trong tương lai

`docker update` ở phần 2 sửa các container đang tồn tại. Nếu sau này Compose tạo lại app, file hiện tại vẫn có thể đặt lại policy thành `unless-stopped`. Nên sửa cấu hình nguồn một lần.

### Bước 1: Sao lưu file

```bash
cp -a compose.yaml "compose.yaml.bak.$(date +%Y%m%d_%H%M%S)"
```

### Bước 2: Sửa cấu hình app backend

```bash
nano compose.yaml
```

Ở đầu file, phần hiện tại bắt đầu bằng `x-app: &app`. Giữ các dòng khác nhưng đổi/thêm hai thuộc tính để phần đầu trông như sau:

```yaml
x-app: &app
  init: true
  restart: "no"
  profiles: ["app"]
  networks:
    - backend
```

Lưu ý:

- Phải viết `restart: "no"` có dấu nháy.
- Không đổi ba dòng `restart: unless-stopped` của `redis`, `rabbitmq`, `payment-postgres`.
- Profile `app` làm cho `docker compose up -d` mặc định chỉ chọn ba service hạ tầng.
- `restart: "no"` cũng có nghĩa app sẽ không tự khởi động lại nếu bị crash trong lúc bạn đang chạy thủ công.

Kiểm tra cú pháp và danh sách service mặc định:

```bash
docker compose config -q
docker compose config --services
docker compose --profile app config --services
```

Lệnh thứ hai chỉ nên liệt kê `redis`, `rabbitmq`, `payment-postgres`. Lệnh thứ ba sẽ liệt kê cả các app.

### Bước 3: Giữ nguyên ý nghĩa của hai npm script Docker

Sau khi thêm profile, hai script hiện có trong `package.json` phải kích hoạt profile `app` nếu bạn vẫn muốn chúng điều khiển toàn bộ stack.

```bash
nano package.json
```

Thay đúng hai script bằng:

```json
"docker:up": "docker compose --profile app up -d --build --wait",
"docker:down": "docker compose --profile app down"
```

Kiểm tra file JSON hợp lệ:

```bash
node -e "JSON.parse(require('node:fs').readFileSync('package.json', 'utf8')); console.log('package.json: OK')"
```

Nếu không sửa hai dòng này, `npm run docker:up` sau khi thêm profile sẽ chỉ chạy hạ tầng, còn `npm run docker:down` có thể không xử lý các app thuộc profile.

### Bước 4: Sửa cấu hình Loki, Alloy, Grafana

Sao lưu file trước:

```bash
cp -a logger/compose.yaml \
  "logger/compose.yaml.bak.$(date +%Y%m%d_%H%M%S)"
```

```bash
nano logger/compose.yaml
```

Trong từng service `loki`, `alloy`, `grafana`, đổi:

```yaml
restart: unless-stopped
```

thành:

```yaml
restart: "no"
```

Kiểm tra cú pháp:

```bash
GRAFANA_ADMIN_PASSWORD=validate-only \
GRAFANA_ALERT_WEBHOOK_URL=https://example.invalid \
  docker compose -f logger/compose.yaml config -q
```

Hai giá trị trên chỉ tồn tại trong đúng lệnh kiểm tra cú pháp; chúng không sửa container và không được ghi vào file.

Phần YAML bảo vệ những lần tạo container trong tương lai. Các container hiện có đã được đồng bộ nếu bạn đã chạy phần 2; nếu chưa, hãy chạy phần 2 một lần.

## 4. Các lệnh sử dụng hằng ngày

Chỉ chạy Redis, RabbitMQ, PostgreSQL:

```bash
cd /media/thanhle/D1/pj1/backend
npm run infra:up
```

Dừng ba service hạ tầng khi không cần:

```bash
npm run infra:down
```

Với policy `unless-stopped`, nếu bạn chủ động chạy `infra:down` thì ba container này cũng sẽ không tự lên ở lần reboot kế tiếp. Chạy lại `npm run infra:up` để bật chúng và khôi phục trạng thái chạy trước khi reboot.

Sau khi đã thêm profile `app`, chạy toàn bộ app bằng Docker:

```bash
docker compose --profile app up -d --build --wait
```

Dừng riêng toàn bộ app, vẫn giữ ba service hạ tầng:

```bash
docker compose --profile app stop \
  auth user mail chat todo workschedule canteen payment gateway
```

Chạy riêng một app khi cần, ví dụ Payment:

```bash
docker compose up -d payment
```

Service được gọi đích danh vẫn chạy được dù thuộc profile `app`; Compose cũng khởi động các dependency cần thiết.

## 5. Lệnh không được dùng nếu muốn giữ dữ liệu

Không chạy các lệnh sau trong quy trình này:

```text
docker compose down -v
docker volume rm nrapp-backend_payment_postgres_data
docker volume prune
```

`docker stop`, `docker update`, và `docker compose up --force-recreate` như hướng dẫn ở trên không xóa named volume PostgreSQL.

## 6. Nếu app vẫn tự chạy sau reboot

Kiểm tra policy thực tế:

```bash
docker ps -a -q | xargs -r docker inspect \
  --format '{{.Name}} -> {{.HostConfig.RestartPolicy.Name}}'
```

Kiểm tra có container nào ngoài hai project trên hay không:

```bash
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Nếu một app có lại `restart=unless-stopped`, thường là do app đã được tạo lại từ một file Compose chưa đổi `restart: "no"`. Sửa file Compose tương ứng rồi chạy lại phần 2.
