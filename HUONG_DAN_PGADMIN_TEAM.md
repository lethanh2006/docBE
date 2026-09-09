# pgAdmin web cho team

Máy host, Docker và `nrapp-cloudflared` phải chạy khi team truy cập. Team chỉ cần
trình duyệt, không cần cài pgAdmin desktop hoặc cloudflared trên từng máy.

## 1. Khởi động

Điền `PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD` (mật khẩu ngẫu nhiên riêng)
và `PGADMIN_HOST_PORT=5050` vào `backend/.env`, theo `backend/.env.example`.

```bash
cd backend
npm run pgadmin:up
docker compose --profile admin ps pgadmin
curl --fail http://127.0.0.1:5050/misc/ping
```

Mở <http://127.0.0.1:5050>, đăng nhập bằng tài khoản trên. Đây là tài khoản web mới,
không phải tài khoản DB hoặc tài khoản pgAdmin desktop.
Volume `pgadmin_data` giữ tài khoản/cấu hình qua restart và recreate. Các biến
`PGADMIN_DEFAULT_*` chỉ tạo admin lần đầu; đổi mật khẩu sau đó trong pgAdmin.
Postfix tắt, chưa cấu hình SMTP nên admin đặt lại mật khẩu thành viên trong
User Management, chưa dùng được chức năng quên mật khẩu qua email.

Service dùng profile `admin`: bật riêng bằng `npm run pgadmin:up`, dừng bằng
`npm run pgadmin:down`, xem log bằng `npm run pgadmin:logs`. `infra:down` dừng DB
nhưng không dừng pgAdmin; `docker:down` dừng cả app và pgAdmin, giữ các volume.
Không dùng `docker compose down -v` nếu cần giữ dữ liệu.

## 2. Cloudflare Access và Tunnel

Tạo Access application trước khi public route:

1. Zero Trust → Access → Applications → Add an application → **Self-hosted**.
2. Hostname: `pgadmin.thanhlelmtp2006.id.vn`, áp dụng toàn bộ đường dẫn.
3. Chọn login bằng One-time PIN qua email hoặc SSO đã cấu hình. Nếu chưa có PIN,
   thêm phương thức này trong phần Login methods của Zero Trust.
4. Policy **Allow** → Include → **Emails** → nhập email của bạn và từng thành viên.
   Không dùng Everyone, Bypass hoặc chỉ Include Login Methods = PIN.
5. Lưu application/policy.

Kiểm tra network của tunnel:

```bash
docker inspect nrapp-cloudflared --format '{{json .NetworkSettings.Networks}}'
```

Nếu thiếu `nrapp-backend_backend`, chạy
`docker network connect nrapp-backend_backend nrapp-cloudflared`. Nếu đổi project
name, lấy tên tương ứng từ `docker network ls`. Khi tạo lại cloudflared cần giữ
hoặc nối lại network này.

Networks → Connectors/Tunnels → tunnel hiện tại → Published application
routes/Public Hostnames → Add:

| Trường | Giá trị |
|---|---|
| Subdomain | `pgadmin` |
| Domain | `thanhlelmtp2006.id.vn` |
| Type | `HTTP` |
| URL | `pgadmin:8080` |

Cũng có thể dùng `nrapp-backend-pgadmin-1:8080` với project name mặc định.
Cổng trong container là **8080** để chạy với `no-new-privileges`; 5050 là cổng host.
Không nhập localhost vào tunnel vì đó là chính container cloudflared.
Trình duyệt dùng HTTPS, tunnel gọi HTTP qua Docker network.

Mở ẩn danh <https://pgadmin.thanhlelmtp2006.id.vn>: phải gặp Access trước pgAdmin.
Thử cả email được Allow và email ngoài danh sách. Access không tự tạo tài khoản
pgAdmin; thành viên vẫn đăng nhập pgAdmin ở bước tiếp theo.

## 3. Tài khoản riêng cho từng người

Admin đăng nhập → Tools → User Management → Add. Chọn authentication `internal`,
email thành viên, role **User**, bật Active, đặt mật khẩu riêng và gửi qua kênh riêng.
Không chia sẻ tài khoản admin. Role User chỉ giới hạn quản trị pgAdmin; quyền sửa
dữ liệu phải giới hạn bằng role PostgreSQL.

Mở psql trên máy host với user sở hữu DB, cũng là user chạy migration hiện tại:

```bash
docker compose exec payment-postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Tạo nhóm quyền **một lần**, cho schema `public` của Payment:

```sql
CREATE ROLE nrapp_payment_readonly NOLOGIN;
SELECT format('GRANT CONNECT ON DATABASE %I TO nrapp_payment_readonly', current_database()) \gexec
GRANT USAGE ON SCHEMA public TO nrapp_payment_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO nrapp_payment_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO nrapp_payment_readonly;
```

Trên máy hiện tại, nhóm `nrapp_payment_readonly` đã được tạo và cấp quyền khi
triển khai ngày 2026-09-06; bỏ qua khối tạo nhóm nếu vẫn dùng DB này. Chưa có
role đăng nhập của thành viên nào được tạo.

Nhóm này xem tất cả bảng Payment trong public. Default privileges áp dụng cho
bảng mới do user đang chạy lệnh tạo; nếu đổi user migration cần cấp lại cho user đó.
Nhóm không cấp quyền ghi; kiểm tra thêm quyền PUBLIC/quyền có sẵn nếu dùng role
hoặc schema đã tùy chỉnh trước đó.

Với mỗi người, thay `team_an` bằng tên riêng:

```sql
CREATE ROLE team_an LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION NOBYPASSRLS;
\password team_an
GRANT nrapp_payment_readonly TO team_an;
```

`\password` nhập mật khẩu ẩn, tránh ghi vào SQL/shell history.
Không đưa credential `PAYMENT_POSTGRES_USER` của ứng dụng cho người chỉ xem.

## 4. Thành viên đăng ký kết nối trong pgAdmin

Đăng nhập bằng tài khoản riêng → Register → Server:

| Trường | Giá trị |
|---|---|
| Name | `Payment (read only)` |
| Host name/address | `payment-postgres` |
| Port | `5432` |
| Maintenance database | Giá trị `PAYMENT_POSTGRES_DB` trong backend/.env |
| Username | Role PostgreSQL riêng, ví dụ `team_an` |
| Password | Mật khẩu đã đặt bằng `\password` |

Mỗi người đăng ký một lần. Không dùng localhost hoặc cổng host 5433.
Mở Schemas → public → Tables → chọn bảng → View/Edit Data → First 100 Rows.
Tài khoản chỉ đọc không lưu được thay đổi dữ liệu.

Khi thu hồi quyền: bỏ email khỏi Access, thu hồi phiên Access đang hoạt động,
tắt Active trong pgAdmin và chạy `ALTER ROLE team_an NOLOGIN;` trong PostgreSQL.
NOLOGIN chặn kết nối mới; ngắt kết nối hiện tại bằng:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity WHERE usename = 'team_an' AND pid <> pg_backend_pid();
```

## Xử lý pgAdmin báo 502

Nếu trình duyệt báo không tìm thấy domain hoặc curl báo `Could not resolve host`,
kiểm tra hostname `pgadmin` còn trong Published application routes/Public
Hostnames và bản ghi DNS còn tồn tại. Nếu đã xóa route, thêm lại theo bước 2;
restart pgAdmin hoặc build backend không tạo lại route/DNS trên Cloudflare.

Nếu `http://127.0.0.1:5050/misc/ping` vẫn trả HTTP 200 nhưng domain báo 502,
kiểm tra route trong Cloudflare. Log `originService=https://...:8080` kèm
`TLS handshake timeout` nghĩa là tunnel đang dùng HTTPS tới cổng HTTP của pgAdmin.

Sửa Published application route/Public Hostname `pgadmin` thành **Type HTTP**,
**URL `pgadmin:8080`**, rồi lưu. URL trên trình duyệt vẫn là
`https://pgadmin.thanhlelmtp2006.id.vn`. Không cần build lại backend hay đổi
chứng chỉ pgAdmin cho cấu hình này.

```bash
docker logs --since=5m --tail=30 nrapp-cloudflared
curl --fail http://127.0.0.1:5050/misc/ping
```

## Tài liệu đối chiếu

- [pgAdmin container](https://www.pgadmin.org/docs/pgadmin4/9.17/container_deployment.html)
- [pgAdmin user management](https://www.pgadmin.org/docs/pgadmin4/9.17/user_management.html)
- [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)
- [PostgreSQL privileges](https://www.postgresql.org/docs/17/ddl-priv.html)
