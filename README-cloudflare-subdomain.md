# Hướng dẫn gắn subdomain qua Cloudflare Tunnel

Áp dụng cho tunnel `nrapp-cloudflared` (chạy theo kiểu **token-based**, quản lý qua Cloudflare Zero Trust Dashboard — không dùng file `config.yml` local).

- Domain gốc: `thanhlelmtp2006.id.vn`
- Đã gắn: `api.thanhlelmtp2006.id.vn` → Swagger/Gateway

---

## 1. Điều kiện bắt buộc: cloudflared phải cùng network với service

Container `cloudflared` gọi service bằng **tên container** qua Docker network nội bộ, nên nó phải nằm chung network `backend` và `observability` với các service muốn public.

Kiểm tra network hiện tại của cloudflared:

```bash
docker inspect nrapp-cloudflared --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

Nếu thiếu network nào, connect thêm (không cần restart container):

```bash
docker network connect nrapp-backend_backend nrapp-cloudflared
docker network connect nrapp-observability nrapp-cloudflared
```

> Tên network chính xác lấy từ `docker network ls` (có thể có prefix khác tuỳ project name).

---

## 2. Truy cập Cloudflare Zero Trust Dashboard

1. Vào **https://one.dash.cloudflare.com/**
2. Chọn account đang chứa tunnel
3. Vào **Networks → Tunnels**
4. Chọn tunnel tương ứng với container `nrapp-cloudflared` (khớp theo token đang chạy)
5. Vào tab **Public Hostname → Add a public hostname**

Với mỗi service HTTP, điền:

| Trường | Giá trị |
|---|---|
| Subdomain | ví dụ `grafana` |
| Domain | `thanhlelmtp2006.id.vn` |
| Type | `HTTP` |
| URL | `<tên_container>:<port>` |

Cloudflare tự tạo DNS CNAME, **không cần** sửa gì ở phần DNS thủ công. Sau khi lưu, thay đổi có hiệu lực sau vài giây tới vài phút, **không cần restart container**.

---

## 3. Bảng mapping subdomain cho từng service

### Backend app services

| Subdomain đề xuất | Container | Port | Ghi chú |
|---|---|---|---|
| `api` | `nrapp-backend-gateway-1` | 3000 | Đã gắn |
| `auth` | `nrapp-backend-auth-1` | 4000 | Nếu muốn public riêng |
| `mail` | `nrapp-backend-mail-1` | 5001 | Thường không cần public |
| `chat` | `nrapp-backend-chat-1` | 5002 | |
| `todo` | `nrapp-backend-todo-1` | 5003 | |
| `workschedule` | `nrapp-backend-workschedule-1` | 5004 | |
| `canteen` | `nrapp-backend-canteen-1` | 5005 | |
| `payment` | `nrapp-backend-payment-1` | 5006 | |

> Lưu ý: các service này (trừ gateway) là internal microservice, gọi lẫn nhau qua `USER_INTERNAL_SECRET`, `PAYMENT_INTERNAL_SECRET`... Thường **không cần** public trực tiếp — client chỉ nên gọi qua `gateway`. Chỉ public thêm nếu bạn có lý do cụ thể (debug, webhook riêng...).

### Observability / quản trị (HTTP — public được qua Tunnel bình thường)

| Subdomain đề xuất | Container | Port | Có auth mặc định? |
|---|---|---|---|
| `grafana` | `nrapp-observability-grafana-1` | 3000 | ✅ Có (login Grafana) |
| `jaeger` | `nrapp-observability-jaeger-1` | 16686 | ❌ Không |
| `prometheus` | `nrapp-observability-prometheus-1` | 9090 | ❌ Không |
| `alertmanager` | `nrapp-observability-alertmanager-1` | 9093 | ❌ Không |
| `rabbitmq` | `nrapp-backend-rabbitmq-1` | 15672 | ✅ Có (login RabbitMQ) |

### PostgreSQL (`payment-postgres`) — TRƯỜNG HỢP ĐẶC BIỆT

PostgreSQL nói chuyện bằng giao thức TCP thuần (wire protocol), **không phải HTTP**, nên không thể gắn kiểu `HTTP Public Hostname` như các dashboard ở trên. Cloudflare Tunnel vẫn hỗ trợ TCP, nhưng theo cách khác:

**Cách 1 — Public Hostname loại TCP (khuyến nghị, có Cloudflare Access bảo vệ)**

Trong Dashboard, khi Add a public hostname, chọn:

| Trường | Giá trị |
|---|---|
| Subdomain | `postgres` |
| Domain | `thanhlelmtp2006.id.vn` |
| Type | `TCP` |
| URL | `nrapp-backend-payment-postgres-1:5432` |

Nhưng: để **kết nối** vào TCP hostname này, máy client (ví dụ laptop bạn dùng để mở DBeaver/pgAdmin) phải cài **`cloudflared` local** và chạy:

```bash
cloudflared access tcp --hostname postgres.thanhlelmtp2006.id.vn --url localhost:5432
```

Sau đó kết nối DB tool vào `localhost:5432` như bình thường. Đây là cách an toàn nhất vì bắt buộc phải qua Cloudflare Access (xác thực) mới connect được, không lộ port DB ra internet trần trụi.

**Cách 2 — Không public, chỉ SSH tunnel khi cần**

Nếu ít khi cần truy cập Postgres từ xa, đơn giản hơn là dùng SSH tunnel tạm thời thay vì cấu hình Cloudflare:

```bash
ssh -L 5433:127.0.0.1:5433 thanhle@<VPS_IP>
```

Rồi connect DB tool vào `localhost:5433` trên máy local.

> ⚠️ Không nên chọn kiểu Public Hostname `HTTP`/để lộ cổng 5432 ra ngoài mà không qua Access — Postgres sẽ nhận traffic tấn công dò mật khẩu ngay lập tức nếu bị bot quét thấy.

---

## 4. Bảo vệ bằng Cloudflare Access (bắt buộc với service không có login)

Với `jaeger`, `prometheus`, `alertmanager`, và `postgres` (TCP) — các service này không có xác thực riêng, nên **bắt buộc** bật Cloudflare Access:

1. Zero Trust → **Access → Applications → Add an application**
2. Chọn loại phù hợp:
   - HTTP service → **Self-hosted**
   - TCP service (Postgres) → tự động áp dụng khi hostname là loại TCP, cấu hình chung trong **Access → Applications**
3. Chọn đúng hostname vừa tạo
4. Tạo **Policy**: yêu cầu login bằng email cụ thể của bạn (One-time PIN qua email) hoặc SSO — chặn hết người khác

Grafana và RabbitMQ đã có login riêng nên có thể bỏ qua Access nếu muốn, nhưng thêm Access vẫn tốt hơn (2 lớp bảo vệ).

---

## 5. Checklist nhanh khi thêm 1 service mới

- [ ] Container có join network `backend`/`observability` chưa? → `docker network connect ...`
- [ ] Thêm Public Hostname trong Zero Trust Dashboard (đúng container:port)
- [ ] Nếu service không có login → thêm Cloudflare Access policy
- [ ] Nếu là TCP (Postgres, Redis, RabbitMQ AMQP...) → dùng type `TCP` + `cloudflared access tcp` phía client, **không** dùng type HTTP
- [ ] Test truy cập từ ngoài bằng `curl -I https://<subdomain>.thanhlelmtp2006.id.vn`
