# Hướng dẫn thực tế triển khai backend NRApp lên VPS

Tài liệu này tổng hợp quy trình đã thực hiện thành công trên VPS NRApp ngày
15/09/2026. Đây là runbook để dựng lại máy mới, kiểm tra hệ thống và truy cập
các trang quản trị.

## 1. Thông tin triển khai hiện tại

| Thành phần | Giá trị |
|---|---|
| VPS | `103.116.52.35` |
| Hostname | `kien24` |
| Hệ điều hành | Ubuntu 22.04.5 LTS, `x86_64` |
| Tài khoản quản trị | `deploy` |
| Thư mục ứng dụng | `/opt/nrapp/backend` |
| Thư mục backup | `/opt/nrapp-backups` |
| API | `https://api-vps.thanhlelmtp2006.id.vn` |
| Swagger | `https://api-vps.thanhlelmtp2006.id.vn/api-docs` |
| Grafana | `https://grafana-vps.thanhlelmtp2006.id.vn` |
| Prometheus | `https://prometheus-vps.thanhlelmtp2006.id.vn` |
| RabbitMQ UI | `https://rabbitmq-vps.thanhlelmtp2006.id.vn` |
| MongoDB | Atlas, database `nrapp` |
| Image release | `vps-lab-001`, kiến trúc `linux/amd64` |

Phạm vi hiện tại:

- Chạy tám ứng dụng: Gateway, Auth, User, Mail, Chat, Todo, Workschedule và
  Canteen.
- Chạy Redis và RabbitMQ.
- Chạy monitoring tối giản gồm Prometheus, Grafana và Node Exporter.
- Tổng cộng 13 container.
- Không chạy Payment, PostgreSQL hay pgAdmin vì Casso đang hết hạn.
- `/api/payment` chủ động trả HTTP 503 với code
  `PAYMENT_TEMPORARILY_DISABLED`.
- Trong giai đoạn này, đơn Canteen chỉ nên dùng `paymentMethod=CASH`.

## 2. Quy ước quan trọng trước khi chạy lệnh

Tài liệu luôn ghi rõ nơi chạy lệnh:

- **Máy cá nhân**: terminal có prompt `thanhle@thanhle`.
- **VPS**: terminal SSH có prompt `deploy@kien24`.
- **Trang web**: thao tác bằng trình duyệt trên MongoDB Atlas, Google,
  Cloudinary hoặc Cloudflare.

Không gửi hoặc chụp màn hình chứa các giá trị sau:

- SSH private key `~/.ssh/nrapp_vps`.
- MongoDB URI hoặc mật khẩu database user.
- Gmail App Password.
- Cloudinary API Secret.
- JWT/internal secret.
- Grafana, RabbitMQ hoặc Nginx Basic Auth password.
- Access token và refresh token của người dùng.

Các file `.env` thật chỉ tồn tại trên VPS, có quyền `600`, không commit Git và
không chuyển ngược về máy khác bằng lệnh không mã hóa.

## 3. Chuẩn bị VPS từ máy trắng

### 3.1. Đăng nhập lần đầu và kiểm tra cấu hình

**Máy cá nhân**:

```bash
ssh root@103.116.52.35
```

**VPS, tài khoản root**:

```bash
cat /etc/os-release
uname -m
uname -r
nproc
free -h
df -h
ip -4 -br addr
timedatectl status
```

Cấu hình thực tế đã dùng là Ubuntu 22.04.5 LTS, 2 vCPU, khoảng 4 GB RAM và
50 GB disk. Không bắt buộc phải nâng lên Ubuntu 24.04 để chạy hệ thống này.

### 3.2. Cập nhật hệ điều hành và công cụ cơ bản

**VPS, root**:

```bash
apt update
apt upgrade -y
apt install -y \
  sudo curl ca-certificates git rsync ufw nginx jq unzip htop \
  dnsutils netcat-openbsd python3 openssl nano less ripgrep

timedatectl set-timezone Asia/Ho_Chi_Minh
timedatectl set-ntp true
timedatectl status
```

Nếu kernel hoặc thư viện hệ thống vừa được nâng cấp:

```bash
sshd -t
reboot
```

Sau reboot, SSH có thể báo `Connection refused` trong một thời gian ngắn. Đợi
máy khởi động và thử lại; không vội cài lại SSH.

### 3.3. Tạo user `deploy`

**VPS, root**:

```bash
adduser deploy
usermod -aG sudo deploy
id deploy
```

Giữ phiên root đang mở cho đến khi đăng nhập bằng `deploy` và chạy được
`sudo whoami`.

### 3.4. Tạo SSH key trên máy cá nhân

**Máy cá nhân**:

```bash
ssh-keygen \
  -t ed25519 \
  -a 100 \
  -f ~/.ssh/nrapp_vps \
  -C "nrapp-vps"
```

Hai file được tạo:

- `~/.ssh/nrapp_vps`: private key, tuyệt đối không gửi cho ai.
- `~/.ssh/nrapp_vps.pub`: public key, được phép chép lên VPS.

Fingerprint chỉ là mã nhận diện key, không phải key dùng để đăng nhập.

Chép public key lên VPS:

```bash
ssh-copy-id \
  -i ~/.ssh/nrapp_vps.pub \
  deploy@103.116.52.35
```

Kiểm tra đăng nhập chỉ bằng key:

```bash
ssh \
  -o IdentitiesOnly=yes \
  -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  -i ~/.ssh/nrapp_vps \
  deploy@103.116.52.35 \
  'whoami && hostname'
```

Kết quả đúng:

```text
deploy
kien24
```

### 3.5. Khóa đăng nhập SSH bằng root/password

**VPS, user `deploy`**:

```bash
sudo tee /etc/ssh/sshd_config.d/00-nrapp.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
EOF

sudo sshd -t

sudo sshd -T \
  | rg 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication'

sudo systemctl reload ssh
```

Mở terminal thứ hai và kiểm tra trước khi đóng phiên hiện tại:

```bash
ssh \
  -o IdentitiesOnly=yes \
  -i ~/.ssh/nrapp_vps \
  deploy@103.116.52.35 \
  'echo SSH_OK'
```

### 3.6. Bật firewall

**VPS**:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Chỉ ba cổng 22, 80 và 443 được mở công khai. Các cổng Docker nội bộ phải bind
vào `127.0.0.1` vì Docker published ports có thể không tuân theo UFW như mong
đợi.

### 3.7. Tạo 2 GB swap

Chỉ chạy nếu `/swapfile` chưa tồn tại:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

grep -qF '/swapfile none swap sw 0 0' /etc/fstab \
  || printf '%s\n' '/swapfile none swap sw 0 0' \
     | sudo tee -a /etc/fstab

swapon --show
free -h
sudo ls -lh /swapfile
```

Swap là vùng đệm khi thiếu RAM ngắn hạn, không thay thế cho RAM thật.

## 4. Cài Docker Engine và Docker Compose

Các lệnh sau tự lấy codename hệ điều hành, vì VPS thực tế dùng Ubuntu Jammy.

**VPS**:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

. /etc/os-release

sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: ${VERSION_CODENAME}
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo usermod -aG docker deploy
```

Đăng xuất rồi SSH lại để group `docker` có hiệu lực:

```bash
exit

ssh \
  -o IdentitiesOnly=yes \
  -i ~/.ssh/nrapp_vps \
  deploy@103.116.52.35
```

Kiểm tra:

```bash
id
docker version --format 'Client={{.Client.Version}} Server={{.Server.Version}}'
docker compose version
docker run --rm hello-world
docker ps
systemctl is-enabled docker
systemctl is-active docker
```

## 5. Tạo thư mục và chuyển source

### 5.1. Tạo thư mục trên VPS

**VPS**:

```bash
sudo mkdir -p /opt/nrapp /opt/nrapp-backups
sudo chown deploy:deploy /opt/nrapp /opt/nrapp-backups
chmod 755 /opt/nrapp
chmod 700 /opt/nrapp-backups

ls -ld /opt/nrapp /opt/nrapp-backups
```

### 5.2. Chuyển source bằng rsync

**Máy cá nhân**, chạy tại `/mnt/data/pj1`:

```bash
cd /mnt/data/pj1

rsync -avz \
  -e "ssh -o IdentitiesOnly=yes -i /home/thanhle/.ssh/nrapp_vps" \
  --exclude='node_modules/' \
  --exclude='dist/' \
  --exclude='coverage/' \
  --exclude='.git/' \
  --exclude='.env' \
  --exclude='.env.local' \
  --exclude='*.log' \
  --exclude='logs/' \
  --exclude='logger/observability/alertmanager/secrets/*' \
  --include='*/' \
  --include='.env.example' \
  --exclude='.env.*' \
  backend/ \
  deploy@103.116.52.35:/opt/nrapp/backend/
```

Không thêm `--delete` vào lần chuyển đầu tiên và không chuyển `.env` thật từ
máy local.

### 5.3. Kiểm tra source trên VPS

```bash
cd /opt/nrapp/backend

test -f compose.yaml && echo "compose.yaml: OK"
test -f docker/node-service.Dockerfile && echo "Dockerfile: OK"
test -f logger/packages/observability/package-lock.json \
  && echo "Logger lockfile: OK"

for service in gateway auth user mail chat todo workschedule canteen; do
  test -f "$service/package-lock.json" \
    && printf '%s: OK\n' "$service" \
    || printf '%s: THIEU package-lock.json\n' "$service"
done

find /opt/nrapp/backend -type f \
  \( -name '.env' -o -name '.env.local' \) -print

du -sh /opt/nrapp/backend
```

Lệnh `find` phải chưa thấy `.env` thật.

## 6. Tạo `.env` và secret nội bộ

### 6.1. Tạo file từ mẫu

**VPS**:

```bash
cd /opt/nrapp/backend
umask 077

test -f .env || cp .env.example .env
test -f logger/.env || cp logger/.env.example logger/.env

for service in gateway auth user mail chat todo workschedule canteen; do
  test -f "$service/.env" \
    || cp "$service/.env.example" "$service/.env"
done

chmod 600 \
  .env logger/.env \
  gateway/.env auth/.env user/.env mail/.env \
  chat/.env todo/.env workschedule/.env canteen/.env

test ! -f payment/.env \
  && echo "Payment env: KHONG TAO - OK"
```

`.env.example` chỉ là khuôn cấu hình. Các database URL và credential thật vẫn
phải cấu hình ở các bước tiếp theo.

### 6.2. Sinh và đồng bộ secret mà không in ra màn hình

Khối này có tính lặp lại an toàn: nếu một secret mạnh đã tồn tại thì giữ lại;
nếu còn placeholder thì mới sinh giá trị mới. Không chạy công cụ khác để in
nội dung `.env` sau bước này.

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from pathlib import Path
from secrets import token_hex

PLACEHOLDERS = (
    "CHANGE_ME", "REPLACE_", "replace_with", "replace-with",
    "your_", "your-", "guest",
)

def values(path):
    result = {}
    for line in path.read_text().splitlines():
        if "=" in line and not line.lstrip().startswith("#"):
            key, value = line.split("=", 1)
            result[key.strip()] = value.strip()
    return result

def set_value(path, key, value):
    lines = path.read_text().splitlines()
    prefix = key + "="
    replaced = False
    output = []
    for line in lines:
        if line.startswith(prefix):
            if not replaced:
                output.append(prefix + value)
                replaced = True
            continue
        output.append(line)
    if not replaced:
        output.append(prefix + value)
    path.write_text("\n".join(output) + "\n")
    path.chmod(0o600)

def secret(path, key):
    current = values(path).get(key, "")
    weak = len(current) < 32 or any(x.lower() in current.lower() for x in PLACEHOLDERS)
    if weak:
        current = token_hex(32)
        set_value(path, key, current)
    return current

root = Path(".env")
gateway = Path("gateway/.env")
auth = Path("auth/.env")
user = Path("user/.env")
chat = Path("chat/.env")
todo = Path("todo/.env")
workschedule = Path("workschedule/.env")
canteen = Path("canteen/.env")
logger = Path("logger/.env")

fixed_root = {
    "COMPOSE_PROJECT_NAME": "nrapp-backend",
    "NRAPP_IMAGE_TAG": "vps-lab-001",
    "OBSERVABILITY_COMPOSE_PROJECT_NAME": "nrapp-observability",
    "OBSERVABILITY_NETWORK_NAME": "nrapp-observability",
    "GATEWAY_BIND_IP": "127.0.0.1",
    "GATEWAY_HOST_PORT": "3000",
    "RABBITMQ_USER": "nrapp_lab",
    "PAYMENT_POSTGRES_USER": "nrapp_payment_disabled",
    "PAYMENT_POSTGRES_DB": "nrapp_payment_disabled",
    "OTEL_TRACES_SAMPLER_ARG": "0",
    "OTEL_METRIC_EXPORT_INTERVAL": "15000",
    "OTEL_METRIC_EXPORT_TIMEOUT": "10000",
}
for key, value in fixed_root.items():
    set_value(root, key, value)

root_secrets = {}
for key in (
    "RABBITMQ_PASSWORD",
    "PAYMENT_POSTGRES_PASSWORD",
    "USER_INTERNAL_SECRET",
    "CHAT_INTERNAL_SECRET",
    "TODO_INTERNAL_SECRET",
    "WORKSCHEDULE_INTERNAL_SECRET",
    "CANTEEN_INTERNAL_SECRET",
    "PAYMENT_INTERNAL_SECRET",
):
    root_secrets[key] = secret(root, key)

jwt_secret = secret(gateway, "JWT_SECRET")
auth_secret = secret(gateway, "AUTH_INTERNAL_SECRET")

for path in (auth, user, chat):
    set_value(path, "JWT_SECRET", jwt_secret)

set_value(auth, "AUTH_INTERNAL_SECRET", auth_secret)

sync = {
    "USER_INTERNAL_SECRET": (gateway, user, todo, workschedule),
    "CHAT_INTERNAL_SECRET": (gateway, chat),
    "TODO_INTERNAL_SECRET": (gateway, todo),
    "WORKSCHEDULE_INTERNAL_SECRET": (gateway, workschedule),
    "CANTEEN_INTERNAL_SECRET": (gateway, canteen),
    "PAYMENT_INTERNAL_SECRET": (gateway,),
}
for key, paths in sync.items():
    for path in paths:
        set_value(path, key, root_secrets[key])

set_value(logger, "GRAFANA_ADMIN_USER", "admin")
secret(logger, "GRAFANA_ADMIN_PASSWORD")
set_value(logger, "OBSERVABILITY_NETWORK_NAME", "nrapp-observability")

for path in (gateway, auth, user, chat, todo, workschedule, canteen):
    set_value(path, "NODE_ENV", "production")
    set_value(path, "LOG_LEVEL", "info")

print("SECRET_SYNC: OK")
print("Khong hien thi bat ky secret nao.")
PY
```

Các biến Payment ở root env chỉ là chốt cấu hình để Compose có thể nội suy;
chúng không làm Payment/PostgreSQL chạy.

## 7. Cấu hình MongoDB Atlas

### 7.1. Tạo cluster và database user trên web

Trên [MongoDB Atlas](https://cloud.mongodb.com/):

1. Tạo project riêng, ví dụ `NRApp VPS Lab`.
2. Tạo cluster `nrapp-vps-lab`.
3. Vào **Security → Database Access → Add New Database User**.
4. Chọn phương thức **Password**.
5. Username: `nrapp_vps_app`.
6. Sinh mật khẩu mạnh và lưu trong password manager.
7. Không chọn `Atlas Admin`, `Read and write to any database` hoặc
   `Only read any database`.
8. Trong **Specific Privileges**, chọn role `readWrite`, database `nrapp`, để
   collection trống.
9. Nếu có mục giới hạn resource, chỉ cho phép cluster `nrapp-vps-lab`.
10. Kết quả phải hiển thị `readWrite@nrapp`.

Nếu Atlas tự tạo một user quyền `Atlas Admin`, hãy xóa user đó sau khi user
`nrapp_vps_app` hoạt động. Nếu mật khẩu từng xuất hiện trong ảnh/chat, coi như
đã lộ và đổi/xóa credential đó.

### 7.2. Chỉ cho IP VPS truy cập Atlas

Vào **Security → Network Access → IP Access List → Add IP Address**:

```text
IP:      103.116.52.35/32
Comment: NRApp VPS
```

Không dùng `0.0.0.0/0`. Xóa IP máy cá nhân tự động được thêm nếu không cần
dùng Compass từ máy đó.

### 7.3. Lấy connection string

Tại cluster, chọn **Connect → Drivers → Node.js**, rồi copy URI mẫu dạng:

```text
mongodb+srv://nrapp_vps_app:<db_password>@...mongodb.net/?...
```

Không gửi URI hoặc mật khẩu qua chat.

### 7.4. Ghi URI vào sáu service bằng prompt ẩn

**VPS**:

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from getpass import getpass
from pathlib import Path
from urllib.parse import quote, urlsplit, urlunsplit, parse_qsl, urlencode

template = getpass("Dan URI mau Atlas vao day (se bi an): ").strip()
password = getpass("Dan mat khau nrapp_vps_app (se bi an): ").strip()

placeholder = next(
    (item for item in ("<db_password>", "<password>") if item in template),
    None,
)
if placeholder is None:
    raise SystemExit("LOI: URI phai con <db_password> hoac <password>")

uri = template.replace(placeholder, quote(password, safe=""), 1)
parts = urlsplit(uri)

if parts.scheme != "mongodb+srv":
    raise SystemExit("LOI: URI khong bat dau bang mongodb+srv://")
if not parts.hostname or not parts.hostname.endswith(".mongodb.net"):
    raise SystemExit("LOI: Khong nhan ra dia chi MongoDB Atlas")

query = [
    (key, value)
    for key, value in parse_qsl(parts.query, keep_blank_values=True)
    if key not in {"maxPoolSize", "minPoolSize"}
]
query.extend([("maxPoolSize", "10"), ("minPoolSize", "0")])

uri = urlunsplit((
    parts.scheme,
    parts.netloc,
    "/nrapp",
    urlencode(query),
    "",
))

services = [
    Path("auth/.env"), Path("user/.env"), Path("chat/.env"),
    Path("todo/.env"), Path("workschedule/.env"), Path("canteen/.env"),
]

def set_env(path, key, value):
    lines = path.read_text().splitlines()
    prefix = key + "="
    for index, line in enumerate(lines):
        if line.startswith(prefix):
            lines[index] = prefix + value
            break
    else:
        lines.append(prefix + value)
    path.write_text("\n".join(lines) + "\n")
    path.chmod(0o600)

for env_file in services:
    set_env(env_file, "MONGO_URL", uri)

set_env(Path("auth/.env"), "MONGO_DB_NAME", "nrapp")
set_env(Path("user/.env"), "MONGO_DB_NAME", "nrapp")

for env_file in services:
    print(f"{env_file}: MONGO_URL OK")
print("Database: nrapp")
PY
```

### 7.5. Kiểm tra Atlas từ VPS

```bash
cd /opt/nrapp/backend

docker pull mongo:8.0

docker run --rm \
  --env-file auth/.env \
  --entrypoint sh \
  mongo:8.0 \
  -lc 'mongosh "$MONGO_URL" --quiet --eval "db.runCommand({ ping: 1 }).ok"'
```

Kết quả đúng là `1`.

Nếu Docker pull báo đã chọn IPv6 nhưng VPS không có đường IPv6, xem mục xử lý
lỗi ở cuối tài liệu.

## 8. Cấu hình Gmail SMTP

### 8.1. Tạo App Password trên web

1. Chọn Gmail dùng để gửi OTP.
2. Bật **2-Step Verification**.
3. Mở [Google App Passwords](https://myaccount.google.com/apppasswords).
4. Tạo app tên `NRApp VPS Mail`.
5. Lưu App Password 16 ký tự.

Google hiển thị mật khẩu thành bốn nhóm có khoảng trắng; script dưới tự bỏ
khoảng trắng. Không dùng mật khẩu đăng nhập Gmail thông thường.

### 8.2. Ghi SMTP vào `mail/.env`

**VPS**:

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from getpass import getpass
from pathlib import Path

email = getpass("Dan dia chi Gmail gui thu (se bi an): ").strip()
app_password = "".join(
    getpass("Dan App Password Google (se bi an): ").split()
)

if "@" not in email or " " in email:
    raise SystemExit("LOI: Dia chi email khong hop le")
if len(app_password) != 16:
    raise SystemExit("LOI: App Password phai co 16 ky tu")

path = Path("mail/.env")
lines = path.read_text().splitlines()
settings = {
    "SMTP_HOST": "smtp.gmail.com",
    "SMTP_PORT": "465",
    "SMTP_SECURE": "true",
    "SMTP_CONNECTION_TIMEOUT_MS": "10000",
    "SMTP_USER": email,
    "SMTP_PASS": app_password,
    "MAIL_FROM": email,
}

for key, value in settings.items():
    prefix = key + "="
    for index, line in enumerate(lines):
        if line.startswith(prefix):
            lines[index] = prefix + value
            break
    else:
        lines.append(prefix + value)

path.write_text("\n".join(lines) + "\n")
path.chmod(0o600)
print("SMTP_CONFIG: OK")
PY
```

### 8.3. Kiểm tra TLS và xác thực, không gửi thư

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
import smtplib
import ssl
from pathlib import Path

values = {}
for line in Path("mail/.env").read_text().splitlines():
    if "=" in line and not line.lstrip().startswith("#"):
        key, value = line.split("=", 1)
        values[key.strip()] = value.strip()

with smtplib.SMTP_SSL(
    values["SMTP_HOST"],
    int(values["SMTP_PORT"]),
    timeout=20,
    context=ssl.create_default_context(),
) as smtp:
    smtp.ehlo()
    smtp.login(values["SMTP_USER"], values["SMTP_PASS"])

print("SMTP_TLS: OK")
print("SMTP_AUTH: OK")
print("Khong gui email trong bai kiem tra nay.")
PY
```

## 9. Cấu hình Cloudinary

### 9.1. Tạo API key trên web

Trong [Cloudinary Console](https://console.cloudinary.com/):

1. Mở **Dashboard/Product Environment** để lấy `Cloud name`.
2. Mở **Settings → API Keys**.
3. Tạo API key riêng tên `NRApp VPS Chat`.
4. Lưu `Cloud name`, `API Key` và `API Secret`.

`Key Name` chỉ là nhãn; nó không phải `Cloud name`. Không dùng key `Root` nếu
đã tạo key riêng.

### 9.2. Ghi vào `chat/.env`

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from getpass import getpass
from pathlib import Path
import re

cloud_name = getpass("Dan Cloud name, chi dan gia tri (se bi an): ").strip()
api_key = getpass("Dan API key NRApp VPS Chat (se bi an): ").strip()
api_secret = getpass("Dan API secret NRApp VPS Chat (se bi an): ").strip()

if not re.fullmatch(r"[A-Za-z0-9_-]+", cloud_name):
    raise SystemExit("LOI: Cloud name khong hop le")
if not api_key or any(c.isspace() for c in api_key):
    raise SystemExit("LOI: API key khong hop le")
if not api_secret or any(c.isspace() for c in api_secret):
    raise SystemExit("LOI: API secret khong hop le")

path = Path("chat/.env")
lines = path.read_text().splitlines()
settings = {
    "CLOUDINARY_CLOUD_NAME": cloud_name,
    "CLOUDINARY_API_KEY": api_key,
    "CLOUDINARY_API_SECRET": api_secret,
}

for key, value in settings.items():
    prefix = key + "="
    for index, line in enumerate(lines):
        if line.startswith(prefix):
            lines[index] = prefix + value
            break
    else:
        lines.append(prefix + value)

path.write_text("\n".join(lines) + "\n")
path.chmod(0o600)
print("CLOUDINARY_CONFIG: OK")
PY
```

### 9.3. Kiểm tra credential, không upload hoặc xóa ảnh

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from pathlib import Path
from urllib.request import Request, urlopen
from urllib.error import HTTPError, URLError
from urllib.parse import quote
import base64
import json

values = {}
for line in Path("chat/.env").read_text().splitlines():
    if "=" in line and not line.lstrip().startswith("#"):
        key, value = line.split("=", 1)
        values[key.strip()] = value.strip()

credentials = base64.b64encode(
    f'{values["CLOUDINARY_API_KEY"]}:{values["CLOUDINARY_API_SECRET"]}'.encode()
).decode()

url = (
    "https://api.cloudinary.com/v1_1/"
    + quote(values["CLOUDINARY_CLOUD_NAME"], safe="")
    + "/resources/image?max_results=1"
)

request = Request(url, headers={"Authorization": f"Basic {credentials}"})

try:
    with urlopen(request, timeout=20) as response:
        data = json.load(response)
    if "resources" not in data:
        raise SystemExit("LOI: Cloudinary tra ve du lieu khong mong doi")
    print("CLOUDINARY_TLS: OK")
    print("CLOUDINARY_AUTH: OK")
    print("Khong upload hoac xoa anh trong bai kiem tra nay.")
except HTTPError as error:
    raise SystemExit(f"LOI HTTP Cloudinary: {error.code}")
except URLError as error:
    raise SystemExit(f"LOI ket noi Cloudinary: {error.reason}")
PY
```

## 10. Kiểm tra placeholder còn sót

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from pathlib import Path

files = [Path(".env"), Path("logger/.env")]
files += [
    Path(service) / ".env"
    for service in (
        "gateway", "auth", "user", "mail", "chat",
        "todo", "workschedule", "canteen",
    )
]

markers = (
    "REPLACE_", "CHANGE_ME", "replace_with", "replace-with",
    "your_cloudinary", "your-email",
)
errors = 0

for path in files:
    for number, line in enumerate(path.read_text().splitlines(), 1):
        if line.lstrip().startswith("#") or "=" not in line:
            continue
        key, value = line.split("=", 1)
        if path == Path(".env") and key.startswith("PGADMIN_"):
            continue
        if path == Path("gateway/.env") and key == "PAYMENT_INTERNAL_SECRET":
            continue
        if any(marker in value for marker in markers):
            print(f"{path}:{number}: con placeholder tai {key.strip()}")
            errors += 1

if errors:
    raise SystemExit(f"LOI: Con {errors} placeholder")

print("ENV_PLACEHOLDER_CHECK: OK")
print("Khong hien thi bat ky secret nao.")
PY
```

## 11. Cấu hình Compose dành riêng cho VPS

Các file đã được chuẩn bị trong repository:

- [compose.vps.yaml](../backend/compose.vps.yaml)
- [rabbitmq.vps.conf](../backend/docker/rabbitmq.vps.conf)
- [compose.vps-minimal.yaml](../backend/logger/compose.vps-minimal.yaml)
- [prometheus.yaml](../backend/logger/vps-minimal/prometheus.yaml)
- [Grafana datasource](../backend/logger/vps-minimal/grafana/provisioning/datasources/prometheus.yaml)

Nếu source đã được rsync sau khi các file này được tạo thì không cần chuyển
lẻ. Nếu thiếu, chạy trên **máy cá nhân**:

```bash
cd /mnt/data/pj1/backend

rsync -avR \
  -e "ssh -o IdentitiesOnly=yes -i /home/thanhle/.ssh/nrapp_vps" \
  ./compose.vps.yaml \
  ./docker/rabbitmq.vps.conf \
  ./logger/compose.vps-minimal.yaml \
  ./logger/vps-minimal/ \
  deploy@103.116.52.35:/opt/nrapp/backend/
```

### 11.1. Tạo helper `dc`

**VPS**:

```bash
mkdir -p /home/deploy/bin

tee /home/deploy/bin/dc >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd /opt/nrapp/backend
exec docker compose \
  --env-file .env \
  -f compose.yaml \
  -f compose.vps.yaml \
  "$@"
EOF

chmod 755 /home/deploy/bin/dc

grep -qxF 'export PATH="/home/deploy/bin:$PATH"' ~/.bashrc \
  || printf '%s\n' 'export PATH="/home/deploy/bin:$PATH"' >> ~/.bashrc

export PATH="/home/deploy/bin:$PATH"
```

### 11.2. Tạo helper `mc`

```bash
tee /home/deploy/bin/mc >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd /opt/nrapp/backend
exec docker compose \
  --project-name nrapp-monitoring-minimal \
  --env-file .env \
  --env-file logger/.env \
  -f logger/compose.vps-minimal.yaml \
  "$@"
EOF

chmod 755 /home/deploy/bin/mc
```

### 11.3. Kiểm tra Compose trước khi chạy

```bash
cd /opt/nrapp/backend

type dc
type mc

dc --profile app config --quiet \
  && echo "BACKEND_CONFIG: OK"

mc config --quiet \
  && echo "MONITORING_CONFIG: OK"

dc --profile app config --services
mc config --services

test "$(dc --profile app config --services | wc -l)" -eq 10
test "$(mc config --services | wc -l)" -eq 3
```

Backend phải chỉ có Redis, RabbitMQ và tám app. Nếu thấy `payment` hoặc
`payment-postgres`, dừng lại, không chạy `up`.

Không gửi output đầy đủ của `docker compose config` lên nơi công khai vì nó có
thể chứa secret đã nội suy.

## 12. Gateway sau Nginx và IP thật của client

Trong [gateway/src/main.ts](../backend/gateway/src/main.ts), ngay sau
`NestFactory.create(...)` phải có:

```typescript
// VPS topology: Internet -> host Nginx -> Gateway container.
// Trust exactly one HTTP proxy so request.ip resolves to the real client IP.
app.getHttpAdapter().getInstance().set('trust proxy', 1);
```

Kiểm tra trên máy cá nhân:

```bash
cd /mnt/data/pj1/backend

rg -n "trust proxy" gateway/src/main.ts

npm ci --prefix logger/packages/observability --no-audit --no-fund
npm ci --prefix gateway --no-audit --no-fund
npm --prefix gateway run format:check
npm --prefix gateway test
```

Kết quả ở lần triển khai này là 26/26 test Gateway đạt.

Nếu vừa sửa file, chuyển nó lên VPS:

```bash
rsync -av \
  -e "ssh -o IdentitiesOnly=yes -i /home/thanhle/.ssh/nrapp_vps" \
  ./gateway/src/main.ts \
  deploy@103.116.52.35:/opt/nrapp/backend/gateway/src/main.ts

sha256sum ./gateway/src/main.ts

ssh \
  -o IdentitiesOnly=yes \
  -i /home/thanhle/.ssh/nrapp_vps \
  deploy@103.116.52.35 \
  'sha256sum /opt/nrapp/backend/gateway/src/main.ts'
```

Hai checksum phải giống nhau.

## 13. Build tám image trên máy cá nhân

`.dockerignore` đã loại `**/.env`, nên secret không đi vào build context.

```bash
cd /mnt/data/pj1/backend

(
  set -euo pipefail
  NRAPP_BUILD_TAG=vps-lab-001

  for service in auth user mail chat todo workschedule canteen gateway; do
    printf '\nDang build: %s\n' "$service"

    docker buildx build \
      --platform linux/amd64 \
      --load \
      --file docker/node-service.Dockerfile \
      --build-arg SERVICE_DIR="$service" \
      --tag "nrapp/$service:$NRAPP_BUILD_TAG" \
      .
  done
)
```

Kiểm tra:

```bash
docker image ls \
  --filter 'reference=nrapp/*:vps-lab-001' \
  --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
```

Phải đủ tám image và không có `nrapp/payment`.

### 13.1. Đóng gói và upload image

**Máy cá nhân**:

```bash
cd /mnt/data/pj1/backend

(
  set -euo pipefail
  NRAPP_BUILD_TAG=vps-lab-001
  NRAPP_IMAGES=()

  for service in gateway auth user mail chat todo workschedule canteen; do
    image="nrapp/$service:$NRAPP_BUILD_TAG"
    architecture="$(docker image inspect --format '{{.Architecture}}' "$image")"
    test "$architecture" = "amd64"
    NRAPP_IMAGES+=("$image")
  done

  docker image save "${NRAPP_IMAGES[@]}" \
    | gzip -1 \
    > /tmp/nrapp-images-vps-lab-001.tar.gz

  cd /tmp
  sha256sum nrapp-images-vps-lab-001.tar.gz \
    > nrapp-images-vps-lab-001.sha256
)

ls -lh \
  /tmp/nrapp-images-vps-lab-001.tar.gz \
  /tmp/nrapp-images-vps-lab-001.sha256

scp \
  -o IdentitiesOnly=yes \
  -i /home/thanhle/.ssh/nrapp_vps \
  /tmp/nrapp-images-vps-lab-001.tar.gz \
  /tmp/nrapp-images-vps-lab-001.sha256 \
  deploy@103.116.52.35:/opt/nrapp/
```

### 13.2. Kiểm tra và import trên VPS

```bash
cd /opt/nrapp

sha256sum -c nrapp-images-vps-lab-001.sha256
df -h /

set -o pipefail
gzip -dc nrapp-images-vps-lab-001.tar.gz \
  | docker image load

docker image ls \
  --filter 'reference=nrapp/*:vps-lab-001' \
  --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
```

Chỉ sau khi đã đủ tám image mới xóa bản tạm trên VPS. Bản gốc trong `/tmp` của
máy cá nhân vẫn được giữ:

```bash
rm -- \
  /opt/nrapp/nrapp-images-vps-lab-001.tar.gz \
  /opt/nrapp/nrapp-images-vps-lab-001.sha256
```

## 14. Tải và khởi động container theo đúng thứ tự

### 14.1. Tải image hạ tầng

```bash
cd /opt/nrapp/backend

dc pull redis rabbitmq
mc pull
```

### 14.2. Redis và RabbitMQ

```bash
docker network inspect nrapp-observability >/dev/null 2>&1 \
  || docker network create nrapp-observability

dc up -d \
  --wait \
  --wait-timeout 300 \
  redis rabbitmq

dc ps redis rabbitmq
dc exec -T redis redis-cli ping
dc exec -T redis redis-cli CONFIG GET maxmemory maxmemory-policy
dc exec -T rabbitmq rabbitmq-diagnostics -q check_running
dc exec -T rabbitmq rabbitmq-diagnostics alarms
```

Kỳ vọng Redis trả `PONG`, maxmemory `50331648`, policy `noeviction`, RabbitMQ
fully booted và không có alarm.

### 14.3. Monitoring tối giản

```bash
mc up -d \
  --wait \
  --wait-timeout 300

mc ps

curl -fsS http://127.0.0.1:9090/-/ready
curl -fsS http://127.0.0.1:3001/api/health | jq .

curl -fsS http://127.0.0.1:9090/api/v1/targets \
  | jq -r '.data.activeTargets[] | [.labels.job, .health] | @tsv'
```

Phải có đúng Grafana, Prometheus và Node Exporter; target `prometheus` và
`node-exporter` đều `up`.

### 14.4. User và Mail trước Auth

```bash
dc --profile app up -d \
  --no-build \
  --pull never \
  --wait \
  --wait-timeout 300 \
  user mail

curl -fsS http://127.0.0.1:5000/health/ready | jq .
curl -fsS http://127.0.0.1:5001/health | jq .
```

User phải báo MongoDB/RabbitMQ `up`; Mail phải báo RabbitMQ/SMTP `up`.

### 14.5. Auth

```bash
dc --profile app up -d \
  --no-build \
  --pull never \
  --wait \
  --wait-timeout 300 \
  auth

curl -fsS http://127.0.0.1:4000/health | jq .
```

### 14.6. Chat, Todo, Workschedule và Canteen

```bash
dc --profile app up -d \
  --no-build \
  --pull never \
  --wait \
  --wait-timeout 300 \
  chat todo workschedule canteen

for item in \
  "chat 5002" \
  "todo 5003" \
  "workschedule 5004" \
  "canteen 5005"
do
  set -- $item
  printf '\n%s:\n' "$1"
  curl -fsS "http://127.0.0.1:$2/health/ready" | jq .
done
```

### 14.7. Gateway cuối cùng

```bash
dc --profile app up -d \
  --no-build \
  --pull never \
  --wait \
  --wait-timeout 300 \
  gateway

dc --profile app ps
curl -fsS http://127.0.0.1:3000/health | jq .
```

Xác nhận Payment/PostgreSQL không tồn tại:

```bash
for service in payment payment-postgres; do
  container_id="$(
    docker ps -aq \
      --filter label=com.docker.compose.project=nrapp-backend \
      --filter "label=com.docker.compose.service=$service"
  )"

  if [ -z "$container_id" ]; then
    echo "$service: TAT - OK"
  else
    echo "$service: LOI - container ton tai"
  fi
done
```

## 15. Cloudflare DNS cho API

Các hostname tunnel cũ không cần sửa. Chúng vẫn phục vụ môi trường local.

Trong **Cloudflare Dashboard → domain `thanhlelmtp2006.id.vn` → DNS → Records**,
tạo:

```text
Type:         A
Name:         api-vps
IPv4 address: 103.116.52.35
Proxy status: DNS only
TTL:          Auto
```

Không tạo `AAAA`. Giữ `DNS only` vì topology hiện tại là client → Nginx →
Gateway và Gateway tin đúng một HTTP reverse proxy.

Kiểm tra trên máy cá nhân:

```bash
dig +short A api-vps.thanhlelmtp2006.id.vn
dig +short AAAA api-vps.thanhlelmtp2006.id.vn
```

Record A phải trả `103.116.52.35`; AAAA không trả gì.

## 16. Nginx, HTTPS và Payment 503

Các file nguồn:

- [nrapp-api.conf](../backend/docker/nginx/nrapp-api.conf)
- [nrapp-swagger](../backend/docker/nginx/nrapp-swagger)
- [nrapp-swagger-on.conf](../backend/docker/nginx/nrapp-swagger-on.conf)
- [nrapp-swagger-off.conf](../backend/docker/nginx/nrapp-swagger-off.conf)

### 16.1. Chuyển file Nginx lên VPS

**Máy cá nhân**:

```bash
cd /mnt/data/pj1/backend

rsync -av \
  -e "ssh -o IdentitiesOnly=yes -i /home/thanhle/.ssh/nrapp_vps" \
  ./docker/nginx/ \
  deploy@103.116.52.35:/opt/nrapp/backend/docker/nginx/
```

### 16.2. Cài site API và helper Swagger

**VPS**:

```bash
cd /opt/nrapp/backend

sudo install -m 644 \
  docker/nginx/nrapp-api.conf \
  /etc/nginx/sites-available/nrapp-api

sudo install -m 644 \
  docker/nginx/nrapp-swagger-on.conf \
  /etc/nginx/snippets/nrapp-swagger-on.conf

sudo install -m 644 \
  docker/nginx/nrapp-swagger-off.conf \
  /etc/nginx/snippets/nrapp-swagger-off.conf

sudo install -m 755 \
  docker/nginx/nrapp-swagger \
  /usr/local/sbin/nrapp-swagger

sudo ln -sfn \
  /etc/nginx/snippets/nrapp-swagger-off.conf \
  /etc/nginx/snippets/nrapp-swagger-active.conf

sudo ln -sfn \
  /etc/nginx/sites-available/nrapp-api \
  /etc/nginx/sites-enabled/nrapp-api

sudo nginx -t
sudo systemctl enable nginx
sudo systemctl reload nginx
```

Lưu ý: nếu Certbot đã chỉnh file `/etc/nginx/sites-available/nrapp-api`, không
chép đè lại file HTTP nguồn ở bước này. Chỉ dùng bước này khi dựng mới trước
khi cấp certificate.

### 16.3. Kiểm tra HTTP trước certificate

```bash
curl -i \
  -H 'Host: api-vps.thanhlelmtp2006.id.vn' \
  http://127.0.0.1/health

curl -i \
  -H 'Host: api-vps.thanhlelmtp2006.id.vn' \
  http://127.0.0.1/api/payment
```

Health phải 200; Payment phải 503.

### 16.4. Cài Certbot và cấp HTTPS

```bash
sudo apt install -y snapd
sudo snap install --classic certbot

if [ ! -e /usr/local/bin/certbot ]; then
  sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
fi

certbot --version

sudo certbot \
  --nginx \
  --redirect \
  -d api-vps.thanhlelmtp2006.id.vn
```

Nhập email quản trị, chấp nhận điều khoản và không cần chia sẻ email quảng cáo.

Kiểm tra:

```bash
sudo nginx -t

curl -fsS \
  https://api-vps.thanhlelmtp2006.id.vn/health \
  | jq .

curl -sS \
  -o /dev/null \
  -w 'HTTP=%{http_code}\nREDIRECT=%{redirect_url}\n' \
  http://api-vps.thanhlelmtp2006.id.vn/health

curl -sS \
  -w '\nPAYMENT_STATUS=%{http_code}\n' \
  https://api-vps.thanhlelmtp2006.id.vn/api/payment

sudo certbot certificates
sudo certbot renew --dry-run
sudo snap services certbot
```

`certbot.renew` ở trạng thái `inactive` nhưng `timer-activated` là bình thường.

## 17. Mở và khóa Swagger

Đường dẫn đúng là `/api-docs`, không phải `/docs-api`.

Mở:

```bash
sudo nrapp-swagger on
```

Khóa và trả 404:

```bash
sudo nrapp-swagger off
```

Kiểm tra:

```bash
curl -sS -o /dev/null \
  -w 'SWAGGER=%{http_code}\n' \
  https://api-vps.thanhlelmtp2006.id.vn/api-docs
```

Khi `on` phải là 200; khi `off` phải là 404. Nếu trình duyệt giữ trang cũ,
nhấn `Ctrl + Shift + R` hoặc mở cửa sổ ẩn danh.

## 18. Subdomain và bảo vệ trang monitoring

Các file nguồn:

- [nrapp-monitoring.conf](../backend/docker/nginx/nrapp-monitoring.conf)
- [nrapp-monitoring](../backend/docker/nginx/nrapp-monitoring)
- [nrapp-monitoring-on.conf](../backend/docker/nginx/nrapp-monitoring-on.conf)
- [nrapp-monitoring-off.conf](../backend/docker/nginx/nrapp-monitoring-off.conf)

### 18.1. Tạo DNS trên Cloudflare

Tạo ba record A, đều trỏ `103.116.52.35`, `DNS only`, TTL Auto:

```text
grafana-vps
prometheus-vps
rabbitmq-vps
```

Không tạo subdomain public cho Node Exporter, Redis hoặc các microservice.

Kiểm tra:

```bash
for host in \
  grafana-vps.thanhlelmtp2006.id.vn \
  prometheus-vps.thanhlelmtp2006.id.vn \
  rabbitmq-vps.thanhlelmtp2006.id.vn
do
  printf '%s: ' "$host"
  dig @1.1.1.1 +short A "$host"
done
```

### 18.2. Tạo Basic Auth và cài Nginx

**VPS**:

```bash
cd /opt/nrapp/backend

sudo apt install -y apache2-utils

sudo htpasswd -c \
  /etc/nginx/.htpasswd-nrapp-monitoring \
  nrappadmin
```

Mật khẩu nhập hai lần, không hiện trên màn hình. Dùng mật khẩu riêng, không
dùng mật khẩu VPS/Grafana/RabbitMQ.

```bash
sudo chown root:www-data \
  /etc/nginx/.htpasswd-nrapp-monitoring

sudo chmod 640 \
  /etc/nginx/.htpasswd-nrapp-monitoring

sudo install -m 644 \
  docker/nginx/nrapp-monitoring.conf \
  /etc/nginx/sites-available/nrapp-monitoring

sudo install -m 644 \
  docker/nginx/nrapp-monitoring-on.conf \
  /etc/nginx/snippets/nrapp-monitoring-on.conf

sudo install -m 644 \
  docker/nginx/nrapp-monitoring-off.conf \
  /etc/nginx/snippets/nrapp-monitoring-off.conf

sudo install -m 755 \
  docker/nginx/nrapp-monitoring \
  /usr/local/sbin/nrapp-monitoring

sudo ln -sfn \
  /etc/nginx/snippets/nrapp-monitoring-on.conf \
  /etc/nginx/snippets/nrapp-monitoring-access.conf

sudo ln -sfn \
  /etc/nginx/sites-available/nrapp-monitoring \
  /etc/nginx/sites-enabled/nrapp-monitoring

sudo nginx -t
sudo systemctl reload nginx
```

Không có credential phải trả HTTP 401, không phải lỗi:

```bash
for host in \
  grafana-vps.thanhlelmtp2006.id.vn \
  prometheus-vps.thanhlelmtp2006.id.vn \
  rabbitmq-vps.thanhlelmtp2006.id.vn
do
  curl -sS -o /dev/null \
    -w "$host: HTTP=%{http_code}\n" \
    "http://$host/"
done
```

### 18.3. Cấp HTTPS cho ba trang

```bash
sudo certbot \
  --nginx \
  --redirect \
  -d grafana-vps.thanhlelmtp2006.id.vn \
  -d prometheus-vps.thanhlelmtp2006.id.vn \
  -d rabbitmq-vps.thanhlelmtp2006.id.vn
```

Kiểm tra không credential; cả ba phải trả 401 qua HTTPS:

```bash
for host in \
  grafana-vps.thanhlelmtp2006.id.vn \
  prometheus-vps.thanhlelmtp2006.id.vn \
  rabbitmq-vps.thanhlelmtp2006.id.vn
do
  curl -sS -o /dev/null \
    -w "$host: HTTPS=%{http_code}\n" \
    "https://$host/"
done
```

### 18.4. Mở/khóa cả ba trang

Mở:

```bash
sudo nrapp-monitoring on
```

Khóa và trả 404:

```bash
sudo nrapp-monitoring off
```

Khi `on`, trình duyệt hỏi Basic Auth:

```text
Username: nrappadmin
Password: mật khẩu đã tạo bằng htpasswd
```

## 19. Đăng nhập từng trang web

### 19.1. Swagger

1. Trên VPS chạy `sudo nrapp-swagger on`.
2. Mở
   [Swagger API](https://api-vps.thanhlelmtp2006.id.vn/api-docs).
3. Test xong chạy `sudo nrapp-swagger off` nếu không cần công khai.

### 19.2. Prometheus

1. Trên VPS chạy `sudo nrapp-monitoring on`.
2. Mở
   [Prometheus](https://prometheus-vps.thanhlelmtp2006.id.vn).
3. Nhập Basic Auth `nrappadmin` và mật khẩu htpasswd.
4. Vào **Status → Target health** để thấy `prometheus` và `node-exporter` đều
   `UP`.

### 19.3. Grafana

Grafana có hai lớp đăng nhập: Basic Auth Nginx rồi đến tài khoản Grafana.

Nếu không nhớ mật khẩu Grafana đã sinh ban đầu, reset bằng prompt ẩn:

```bash
cd /opt/nrapp/backend

python3 - <<'PY'
from getpass import getpass
from pathlib import Path
import subprocess

password = getpass("Mat khau Grafana moi (se bi an): ")
confirm = getpass("Nhap lai mat khau Grafana: ")

if password != confirm:
    raise SystemExit("LOI: Hai mat khau khong khop")
if len(password) < 12:
    raise SystemExit("LOI: Mat khau phai co it nhat 12 ky tu")

subprocess.run([
    "/home/deploy/bin/mc", "exec", "-T", "grafana",
    "grafana", "cli", "--homepath", "/usr/share/grafana",
    "admin", "reset-admin-password", password,
], check=True)

path = Path("logger/.env")
lines = path.read_text().splitlines()
prefix = "GRAFANA_ADMIN_PASSWORD="
for index, line in enumerate(lines):
    if line.startswith(prefix):
        lines[index] = prefix + password
        break
else:
    lines.append(prefix + password)

path.write_text("\n".join(lines) + "\n")
path.chmod(0o600)
print("GRAFANA_PASSWORD_RESET: OK")
print("GRAFANA_USER: admin")
PY
```

Sau đó:

1. Mở [Grafana](https://grafana-vps.thanhlelmtp2006.id.vn).
2. Qua Basic Auth bằng `nrappadmin` và mật khẩu htpasswd.
3. Đăng nhập Grafana bằng username `admin` và mật khẩu Grafana vừa đặt.
4. Datasource Prometheus đã được provision tự động.

### 19.4. RabbitMQ Management UI

Không dùng tài khoản ứng dụng để xem dashboard. Tạo user chỉ giám sát:

```bash
cd /opt/nrapp/backend

(
  set -euo pipefail

  read -rsp 'Mat khau moi cho nrapp_monitor: ' NRAPP_RABBIT_MONITOR_PASSWORD
  echo
  read -rsp 'Nhap lai mat khau: ' NRAPP_RABBIT_MONITOR_PASSWORD_CONFIRM
  echo

  if [ "$NRAPP_RABBIT_MONITOR_PASSWORD" != "$NRAPP_RABBIT_MONITOR_PASSWORD_CONFIRM" ]; then
    echo 'LOI: Hai mat khau khong khop'
    exit 1
  fi

  if dc exec -T rabbitmq rabbitmqctl list_users \
    | awk '$1 == "nrapp_monitor" { found = 1 } END { exit !found }'; then
    dc exec -T rabbitmq rabbitmqctl change_password \
      nrapp_monitor "$NRAPP_RABBIT_MONITOR_PASSWORD"
  else
    dc exec -T rabbitmq rabbitmqctl add_user \
      nrapp_monitor "$NRAPP_RABBIT_MONITOR_PASSWORD"
  fi

  dc exec -T rabbitmq rabbitmqctl set_user_tags \
    nrapp_monitor monitoring

  dc exec -T rabbitmq rabbitmqctl set_permissions \
    --vhost / nrapp_monitor '^$' '^$' '^$'

  dc exec -T rabbitmq rabbitmqctl list_users
)
```

Lưu password `nrapp_monitor` trong password manager. Sau đó:

1. Mở [RabbitMQ UI](https://rabbitmq-vps.thanhlelmtp2006.id.vn).
2. Qua Basic Auth bằng `nrappadmin` và mật khẩu htpasswd.
3. Đăng nhập RabbitMQ bằng username `nrapp_monitor` và password vừa tạo.
4. User có tag `monitoring`, không có quyền publish, consume, tạo hoặc xóa
   queue/exchange.

## 20. Test nghiệp vụ bằng Swagger

Mở Swagger và thực hiện theo thứ tự:

1. `POST /api/auth/register` với email nhận được thư, password NRApp riêng và
   username.
2. `POST /api/auth/login` với email/password để gửi OTP.
3. Lấy OTP sáu chữ số trong email; OTP có hạn năm phút.
4. `POST /api/auth/verify` với email và OTP.
5. Copy `accessToken`; không gửi token qua chat.
6. Bấm **Authorize** trong Swagger và dán access token.
7. Gọi `GET /api/auth/me` để kiểm tra JWT.
8. Test Todo, Workschedule và Chat theo các route hiện trên Swagger.
9. Khi tạo đơn Canteen, dùng `paymentMethod=CASH`.
10. `/api/payment` phải luôn trả HTTP 503 trong release này.

Route `/api/auth/login-google` đang chủ động trả 404 tại Nginx cho đến khi
luồng Google token được kiểm tra và cấu hình đầy đủ.

## 21. Kiểm tra cổng công khai

**Máy cá nhân**:

```bash
for port in \
  22 80 443 \
  3000 3001 4000 \
  5000 5001 5002 5003 5004 5005 \
  5672 6379 9090 15672
do
  if nc -z -w 2 103.116.52.35 "$port" 2>/dev/null; then
    echo "$port: OPEN"
  else
    echo "$port: CLOSED/FILTERED"
  fi
done
```

Chỉ 22, 80 và 443 được `OPEN`; mọi cổng còn lại phải
`CLOSED/FILTERED`.

## 22. Lệnh vận hành thường dùng

### 22.1. Xem trạng thái

```bash
dc --profile app ps
mc ps
docker stats --no-stream
free -h
df -h /
```

### 22.2. Xem log

```bash
dc logs --tail 100 gateway
dc logs --tail 100 auth
dc logs --tail 100 user mail
dc logs --tail 100 chat todo workschedule canteen
dc logs --tail 100 rabbitmq redis
mc logs --tail 100 prometheus grafana node-exporter
```

Theo dõi realtime một service:

```bash
dc logs -f --tail 100 gateway
```

Nhấn `Ctrl+C` chỉ thoát chế độ xem log, không dừng container.

### 22.3. Restart một service

```bash
dc restart gateway
dc restart auth
mc restart grafana
```

### 22.4. Bật lại sau khi bảo trì

```bash
cd /opt/nrapp/backend

docker network inspect nrapp-observability >/dev/null 2>&1 \
  || docker network create nrapp-observability

dc up -d --wait --wait-timeout 300 redis rabbitmq
mc up -d --wait --wait-timeout 300

dc --profile app up -d \
  --no-build --pull never \
  --wait --wait-timeout 300 \
  user mail auth chat todo workschedule canteen gateway
```

### 22.5. Không dùng tùy tiện

Không chạy các lệnh sau nếu chưa có backup và chưa hiểu ảnh hưởng:

```text
docker compose down -v
docker system prune --volumes
docker volume rm ...
dc --profile payment-later up ...
```

`-v` có thể xóa volume Redis, RabbitMQ, Grafana hoặc Prometheus. Không bật
profile `payment-later` trong giai đoạn Casso hết hạn.

## 23. Các lỗi đã gặp và cách xử lý

### 23.1. SSH `Connection refused` ngay sau reboot

Đợi VPS khởi động hoàn tất rồi SSH lại. Nếu kéo dài, dùng console của nhà cung
cấp để kiểm tra `systemctl status ssh` và firewall.

### 23.2. Docker pull chọn IPv6 và báo `network is unreachable`

Chỉ dùng cách này khi log thực sự cho thấy Docker/CloudFront đang thử địa chỉ
IPv6 nhưng VPS không có IPv6:

```bash
sudo tee /etc/sysctl.d/99-nrapp-disable-ipv6.conf >/dev/null <<'EOF'
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
EOF

sudo sysctl --system
sudo systemctl restart docker
```

Sau đó chạy lại `docker pull`. Việc restart Docker sẽ restart các container có
restart policy phù hợp; nên kiểm tra `dc ps` và `mc ps` sau đó.

### 23.3. RabbitMQ `ping` được nhưng `rabbit` app chưa chạy

Healthcheck `ping` có thể thành công sớm hơn lúc RabbitMQ boot hoàn toàn. Đợi
thêm rồi kiểm tra:

```bash
dc exec -T rabbitmq rabbitmq-diagnostics -q check_running
dc exec -T rabbitmq rabbitmq-diagnostics alarms
```

Không chạy `rabbitmqctl start_app` để che lỗi khởi động container. Nếu vẫn lỗi:

```bash
dc logs --tail 120 rabbitmq
```

### 23.4. Swagger trả 404

Kiểm tra đúng URL `/api-docs`, không phải `/docs-api`, rồi chạy:

```bash
sudo nrapp-swagger on

curl -sS -o /dev/null \
  -w 'GATEWAY_DIRECT=%{http_code}\n' \
  http://127.0.0.1:3000/api-docs

curl -sS -o /dev/null \
  -w 'NGINX_HTTPS=%{http_code}\n' \
  https://api-vps.thanhlelmtp2006.id.vn/api-docs
```

Nếu Gateway 200 nhưng Nginx 404:

```bash
readlink -f /etc/nginx/snippets/nrapp-swagger-active.conf
sudo nginx -T 2>/dev/null | grep -n -B 2 -A 3 'api-docs'
```

Không để đồng thời một block `location ^~ /api-docs { return 404; }` cũ và
snippet bật/tắt mới.

### 23.5. Monitoring trả 401

Khi `nrapp-monitoring on`, HTTP 401 khi chưa nhập credential là đúng. Dùng
Basic Auth `nrappadmin`. Khi `off`, kết quả đúng là 404.

### 23.6. Container không healthy

```bash
dc --profile app ps
dc logs --tail 150 TEN_SERVICE
docker inspect \
  --format '{{json .State.Health}}' \
  nrapp-backend-TEN_SERVICE-1 \
  | jq .
```

Không restart liên tục trước khi đọc log và xác định dependency nào đang lỗi.

## 24. Checklist nghiệm thu hiện tại

- [x] Ubuntu 22.04.5 LTS đã cập nhật.
- [x] User `deploy`, SSH key-only, root login và password SSH đã khóa.
- [x] UFW chỉ mở 22/80/443.
- [x] Có 2 GB swap.
- [x] Docker Engine và Compose hoạt động.
- [x] MongoDB Atlas `readWrite@nrapp`, chỉ whitelist IP VPS.
- [x] SMTP TLS/Auth hoạt động.
- [x] Cloudinary TLS/Auth hoạt động.
- [x] Tám image `linux/amd64` đã build và import.
- [x] Mười container backend healthy.
- [x] Ba container monitoring healthy.
- [x] Payment/PostgreSQL không được tạo.
- [x] API HTTPS hoạt động và HTTP redirect sang HTTPS.
- [x] Payment trả 503 riêng, không làm sập các service khác.
- [x] Swagger có lệnh bật/tắt.
- [x] Grafana, Prometheus và RabbitMQ có HTTPS, Basic Auth và lệnh bật/tắt.
- [x] Chỉ cổng 22/80/443 truy cập được từ Internet.
- [ ] Hoàn thành/ghi nhớ mật khẩu đăng nhập Grafana sau khi reset.
- [ ] Tạo user RabbitMQ `nrapp_monitor` nếu chưa thực hiện.
- [ ] Test các nghiệp vụ còn lại bằng Swagger theo nhu cầu.

## 25. Tài liệu và file liên quan

- [Kế hoạch VPS đầy đủ](./KE_HOACH_THUE_VPS_VA_TRIEN_KHAI_BACKEND.md)
- [Luồng Canteen và Payment](./GIAI_THICH_LUONG_BACKEND_CANTEEN_PAYMENT.md)
- [Compose VPS](../backend/compose.vps.yaml)
- [Monitoring Compose](../backend/logger/compose.vps-minimal.yaml)
- [Cấu hình Nginx](../backend/docker/nginx/)

## 26. Continuous Deployment từ GitHub Actions

### 26.1. Kiến trúc CD

Mỗi service là một Git repository độc lập. Tám repository đang chạy trên VPS
có workflow `CD` riêng:

| Service | GitHub repository | Nhánh mặc định |
|---|---|---|
| Auth | `lethanh2006/AUTH_SERVICE` | `main` |
| Canteen | `lethanh2006/CANTEEN_SERVICE` | `main` |
| Chat | `lethanh2006/CHAT_SERVICE` | `master` |
| Gateway | `lethanh2006/API-GATEWAY` | `main` |
| Mail | `lethanh2006/MAIL_SERVICE` | `main` |
| Todo | `lethanh2006/TODO_SERVICE` | `main` |
| User | `lethanh2006/USER_SERVICE` | `main` |
| Workschedule | `lethanh2006/WORKSCHEDULE_SERVICE` | `main` |

Payment không có CD vì chưa được triển khai.

Luồng tự động:

1. Push code vào nhánh mặc định.
2. Workflow `CI` chạy lint, test và build.
3. Chỉ khi CI thành công, workflow `CD` checkout đúng commit SHA đã qua CI.
4. GitHub runner build đúng một image `linux/amd64`.
5. Image được nén và truyền trực tiếp qua SSH; không cần Docker registry.
6. VPS lưu image hiện tại thành tag `rollback`, nạp image mới và chỉ recreate
   service vừa thay đổi.
7. Compose chờ healthcheck. Nếu lỗi, receiver tự đưa image cũ trở lại.

Các file của mỗi service:

```text
.github/
├── workflows/cd.yml
├── docker/node-service.Dockerfile
├── docker/node-service.Dockerfile.dockerignore
└── known_hosts
```

Receiver dùng chung được lưu tại:

```text
backend/logger/deploy/vps-ci-receiver
```

Và đã được cài trên VPS thành:

```text
/home/deploy/bin/nrapp-ci-receiver
```

### 26.2. Bảo vệ SSH cho CD

Không dùng private key cá nhân `nrapp_vps`. Mỗi repository có một Ed25519 key
riêng:

```text
~/.ssh/nrapp_github_cd_auth
~/.ssh/nrapp_github_cd_canteen
~/.ssh/nrapp_github_cd_chat
~/.ssh/nrapp_github_cd_gateway
~/.ssh/nrapp_github_cd_mail
~/.ssh/nrapp_github_cd_todo
~/.ssh/nrapp_github_cd_user
~/.ssh/nrapp_github_cd_workschedule
```

Public key đã được cài vào `authorized_keys` bằng `restrict` và forced command.
Ví dụ key của Auth chỉ gọi được:

```text
/home/deploy/bin/nrapp-ci-receiver auth
```

Nó không mở được shell, không forwarding và không deploy được service khác.
Receiver cũng từ chối Payment.

### 26.3. Thêm secret vào tám GitHub repository

Máy hiện tại chưa đăng nhập GitHub CLI. Trên **máy cá nhân**, đăng nhập tài
khoản sở hữu các repository:

```bash
gh auth login
gh auth status
```

Sau đó đặt secret mà không in private key ra màn hình:

```bash
declare -A NRAPP_CD_REPOSITORIES=(
  [auth]='lethanh2006/AUTH_SERVICE'
  [canteen]='lethanh2006/CANTEEN_SERVICE'
  [chat]='lethanh2006/CHAT_SERVICE'
  [gateway]='lethanh2006/API-GATEWAY'
  [mail]='lethanh2006/MAIL_SERVICE'
  [todo]='lethanh2006/TODO_SERVICE'
  [user]='lethanh2006/USER_SERVICE'
  [workschedule]='lethanh2006/WORKSCHEDULE_SERVICE'
)

for service in \
  auth canteen chat gateway mail todo user workschedule
do
  key="$HOME/.ssh/nrapp_github_cd_${service}"
  repository="${NRAPP_CD_REPOSITORIES[$service]}"

  test -s "$key"
  gh secret set \
    VPS_SSH_PRIVATE_KEY \
    --repo "$repository" \
    <"$key"
done
```

Chỉ kiểm tra tên secret, GitHub không cho đọc lại giá trị:

```bash
for repository in "${NRAPP_CD_REPOSITORIES[@]}"; do
  printf '\n%s:\n' "$repository"
  gh secret list --repo "$repository" \
    | rg '^VPS_SSH_PRIVATE_KEY\b'
done
```

Nếu Logger chuyển thành private, vẫn phải cấu hình thêm `LOGGER_READ_TOKEN`
như tài liệu CI của từng repository.

### 26.4. Kích hoạt CD

Workflow chỉ hoạt động sau khi commit chứa `cd.yml` được push lên GitHub. Push
từng repository bằng đúng nhánh hiện tại; không dùng `--force`:

```bash
git -C /mnt/data/pj1/backend/auth push origin main
git -C /mnt/data/pj1/backend/canteen push origin main
git -C /mnt/data/pj1/backend/chat push origin master
git -C /mnt/data/pj1/backend/gateway push origin main
git -C /mnt/data/pj1/backend/mail push origin main
git -C /mnt/data/pj1/backend/todo push origin main
git -C /mnt/data/pj1/backend/user push origin main
git -C /mnt/data/pj1/backend/workschedule push origin main
```

Một push vào nhánh mặc định sẽ tạo hai workflow liên tiếp trong tab
**Actions**: `CI`, rồi đến `CD`. Pull Request, nhánh phụ, Dependabot và CI thất
bại đều không deploy.

### 26.5. Theo dõi và rollback

Kiểm tra lịch sử deployment trên VPS:

```bash
tail -n 30 /opt/nrapp/cd/history.tsv
```

Xem image rollback đang giữ:

```bash
docker image ls \
  --filter 'reference=nrapp/*:rollback' \
  --format 'table {{.Repository}}\t{{.Tag}}\t{{.CreatedSince}}\t{{.Size}}'
```

Xem trạng thái và log sau deployment:

```bash
dc --profile app ps
dc --profile app logs --tail 150 TEN_SERVICE
```

Nếu healthcheck deployment mới thất bại, CD tự rollback và job GitHub vẫn đỏ
để báo cần sửa code. Không xóa tag `rollback` khi chưa kiểm tra release mới.

### 26.6. Trạng thái thiết lập CD

- [x] Có workflow CD cho tám service đang chạy.
- [x] Build image `linux/amd64` trên GitHub runner.
- [x] Receiver có deployment lock, healthcheck và rollback.
- [x] Tám SSH key riêng đã tạo và public key đã cài trên VPS.
- [x] Shell, cross-service deploy và Payment đã bị chặn.
- [x] Host key VPS được pin trong repository.
- [ ] Đăng nhập `gh` và thêm `VPS_SSH_PRIVATE_KEY` vào tám repository.
- [ ] Push các commit CD lên GitHub để chạy deployment đầu tiên.
