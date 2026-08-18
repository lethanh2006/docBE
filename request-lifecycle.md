# Giải thích chi tiết luồng request production của Canteen

Tài liệu này giải thích cách một HTTP request đi từ client qua API Gateway đến
Canteen Service, cách request được xác thực, kiểm tra dữ liệu, xử lý nghiệp vụ,
ghi log và trả response.

Mục tiêu là giúp người mới đọc code trả lời được các câu hỏi:

- Request bắt đầu từ file nào?
- Middleware, guard, interceptor, pipe, controller và service khác nhau ở đâu?
- Vì sao cần `x-request-id`?
- Vì sao Base64 chưa đủ an toàn và phải ký HMAC?
- Lỗi ở từng bước sẽ dừng request như thế nào?
- Log Telegram/Discord nằm ở đâu trong kiến trúc?
- Khi thêm endpoint mới thì nên đặt code vào thư mục nào?

> Middleware, guard, interceptor, pipe và filter là các cơ chế trong vòng đời
> request, không phải tám tầng bắt buộc cho mọi endpoint. Endpoint chỉ sử dụng
> những thành phần cần thiết.

---

## 1. Bức tranh tổng thể

```text
Client
  │
  │ Authorization: Bearer <JWT>
  ▼
API Gateway
  │
  ├─ 1. RequestIdMiddleware
  │     Tạo hoặc nhận x-request-id
  │
  ├─ 2. RateLimitMiddleware
  │     Giới hạn số request theo IP
  │
  ├─ 3. JwtAuthGuard
  │     Xác thực access token với Auth Service
  │
  ├─ 4. RolesGuard
  │     Kiểm tra role tại Gateway
  │
  └─ 5. CanteenService
        ├─ Encode thông tin user thành Base64
        ├─ Ký HMAC payload
        └─ Forward request + request ID đến Canteen
                         │
                         ▼
Canteen Service
  │
  ├─ 6. RequestIdMiddleware
  │     Nhận lại đúng request ID từ Gateway
  │
  ├─ 7. RolesGuard
  │     Xác minh HMAC, đọc user và kiểm tra role lần hai
  │
  ├─ 8. HttpLoggingInterceptor
  │     Bao quanh quá trình xử lý để ghi status và duration
  │
  ├─ 9. ValidationPipe / ParseObjectIdPipe
  │     Kiểm tra body, query và route params
  │
  ├─ 10. Controller
  │      Nhận dữ liệu đã validate và gọi service
  │
  ├─ 11. Domain Service
  │      Xử lý nghiệp vụ; gọi MongoDB, Redis hoặc RabbitMQ
  │
  └─ 12. Response
         Trả dữ liệu và header x-request-id cho client

Nếu có exception trong Canteen:
  GlobalExceptionFilter
    ├─ Xác định status code
    ├─ Ghi structured log
    ├─ Gắn requestId vào error response
    └─ Trả lỗi cho Gateway → Client
```

Một request có thể dừng sớm. Ví dụ rate limit trả `429` ngay tại Gateway thì
request chưa hề tới Canteen. Guard trả `403` thì pipe, controller và service
không chạy.

---

## 2. Cấu trúc thư mục và trách nhiệm

### 2.1. Canteen Service

```text
backend/canteen/src/
├── main.ts
│   ├── Khởi tạo Nest application
│   ├── Đăng ký global ValidationPipe
│   └── Bật graceful shutdown
│
├── core/
│   └── core.module.ts
│       ├── Đăng ký RequestIdMiddleware
│       ├── Đăng ký global HttpLoggingInterceptor
│       ├── Đăng ký GlobalExceptionFilter
│       └── Cung cấp logger và dịch vụ xác minh HMAC
│
├── common/
│   ├── decorators/
│   │   ├── roles.decorator.ts
│   │   ├── authenticated.decorator.ts
│   │   └── user.decorator.ts
│   ├── middleware/
│   │   └── request-id.middleware.ts
│   ├── guards/
│   │   └── roles.guard.ts
│   ├── security/
│   │   └── gateway-signature.service.ts
│   ├── interceptors/
│   │   └── http-logging.interceptor.ts
│   ├── pipes/
│   │   └── parse-object-id.pipe.ts
│   ├── filters/
│   │   └── global-exception.filter.ts
│   ├── observability/
│   │   └── structured-logger.service.ts
│   ├── interfaces/
│   │   ├── authenticated-user.interface.ts
│   │   └── request-context.interface.ts
│   └── crud/
│       └── CRUD dùng chung cho category, table, ingredient...
│
└── modules/
    ├── category/       Controller + Service + DTO của danh mục
    ├── table/          Controller + Service + DTO của bàn ăn
    ├── menu/           Controller + Service + DTO của thực đơn
    ├── order/          Controller + Service + DTO của đơn hàng
    ├── kitchen/        Nghiệp vụ nhà bếp
    ├── inventory/      Nghiệp vụ kho
    ├── analytics/      Nghiệp vụ thống kê
    ├── health/         Liveness và readiness
    ├── database/       Kết nối MongoDB
    ├── redis/          Adapter Redis
    └── rabbitmq/       Adapter RabbitMQ
```

Quy tắc quan trọng:

- `common` chứa cơ chế kỹ thuật dùng chung, không chứa nghiệp vụ căn tin.
- `modules/<domain>` chứa nghiệp vụ của từng domain.
- Controller nên mỏng: nhận input, gọi service và trả output.
- Service không nên biết Express request/response nếu không thực sự cần.
- MongoDB, Redis và RabbitMQ là adapter hạ tầng, không phải controller layer.

### 2.2. Gateway

Các file Gateway liên quan trực tiếp:

```text
backend/gateway/src/
├── core/core.module.ts
├── common/interfaces/request-context.interface.ts
├── common/middleware/request-id.middleware.ts
├── common/middleware/rate-limit.middleware.ts
├── common/security/internal-request-signature.service.ts
└── modules/canteen/
    ├── canteen.controller.ts
    ├── canteen.service.ts
    └── canteen.module.ts
```

Gateway chịu trách nhiệm xác thực JWT từ client. Canteen không nhận JWT trực
tiếp trong thiết kế hiện tại; nó nhận thông tin user đã được Gateway xác thực và
ký lại bằng secret dùng nội bộ.

---

## 3. Vai trò của từng thành phần

| Thành phần            | Câu hỏi nó trả lời                           | Không nên làm                      |
| --------------------- | -------------------------------------------- | ---------------------------------- |
| Request ID middleware | “Đây là lần gọi API nào?”                    | Kiểm tra role hoặc xử lý nghiệp vụ |
| Rate limit middleware | “IP này có gọi quá nhiều không?”             | Xác định user có role gì           |
| JWT guard tại Gateway | “Token có hợp lệ và user còn tồn tại không?” | Cập nhật bàn hoặc tạo đơn          |
| Roles guard           | “User này có quyền vào endpoint không?”      | Validate chi tiết DTO              |
| Interceptor           | “Request mất bao lâu, trả status nào?”       | Chứa business logic                |
| Pipe                  | “Input có đúng kiểu và quy tắc không?”       | Truy vấn database phức tạp         |
| Controller            | “Endpoint này gọi use case nào?”             | Chứa toàn bộ nghiệp vụ             |
| Service               | “Nghiệp vụ phải xử lý như thế nào?”          | Trực tiếp thao tác HTTP response   |
| Exception filter      | “Lỗi được log và trả về theo dạng nào?”      | Che giấu lỗi nghiệp vụ hợp lệ      |
| Monitoring            | “Hệ thống có đang bất thường không?”         | Tham gia đồng bộ vào mọi request   |

---

## 4. Ví dụ đầy đủ: cập nhật trạng thái bàn

Endpoint đi qua Gateway:

```http
PATCH /api/canteen/tables/507f1f77bcf86cd799439011/status
Authorization: Bearer <access-token>
Content-Type: application/json

{
  "status": "occupied"
}
```

Endpoint cho phép `ADMIN`, `MANAGER` hoặc `WAITER`.

### Bước 1: Gateway tạo Request ID

File:

```text
backend/gateway/src/common/middleware/request-id.middleware.ts
```

Nếu client gửi một `x-request-id` hợp lệ, Gateway có thể tiếp tục dùng ID đó.
Nếu không có hoặc định dạng không an toàn, Gateway tạo UUID mới:

```text
x-request-id: 8ee70f08-7f49-4187-b842-76a3cb811670
```

Gateway đồng thời lưu ID vào `request.requestContext`, thêm ID vào request
header và thêm ID vào response header. Nhờ đó client, Gateway và Canteen có thể
cùng nói về một request bằng một mã duy nhất.

### Bước 2: Gateway kiểm tra rate limit

File:

```text
backend/gateway/src/common/middleware/rate-limit.middleware.ts
```

Mặc định mỗi IP được gọi tối đa 120 request trong 60 giây. Response có các
header:

```http
x-ratelimit-limit: 120
x-ratelimit-remaining: 119
x-ratelimit-reset: 1787025600
```

Nếu vượt giới hạn, Gateway trả ngay:

```http
HTTP/1.1 429 Too Many Requests
retry-after: 42
x-request-id: 8ee70f08-7f49-4187-b842-76a3cb811670
```

```json
{
  "statusCode": 429,
  "message": "Quá nhiều request, vui lòng thử lại sau",
  "requestId": "8ee70f08-7f49-4187-b842-76a3cb811670"
}
```

Ở nhánh này JWT guard và Canteen đều không chạy.

### Bước 3: Gateway xác thực JWT và role

`CanteenController` được gắn:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
```

`JwtAuthGuard` dùng Auth Service để introspect token. Nếu token hợp lệ, thông
tin user được gắn vào `request.user`, ví dụ:

```json
{
  "_id": "64b7f0f0f0f0f0f0f0f0f0f0",
  "email": "waiter@example.com",
  "role": "WAITER"
}
```

Gateway `RolesGuard` so sánh role với decorator của endpoint:

- thiếu hoặc hết hạn token → `401 Unauthorized`;
- token hợp lệ nhưng sai role → `403 Forbidden`;
- token và role hợp lệ → chuyển sang Gateway service.

### Bước 4: Gateway ký thông tin user

Các file:

```text
backend/gateway/src/modules/canteen/canteen.service.ts
backend/gateway/src/common/security/internal-request-signature.service.ts
```

Gateway thực hiện:

1. `JSON.stringify(user)`;
2. encode JSON thành Base64;
3. lấy timestamp hiện tại;
4. ký chuỗi `timestamp.requestId.payload` bằng HMAC SHA-256;
5. forward request sang Canteen.

Request nội bộ có các header:

```http
x-request-id: 8ee70f08-7f49-4187-b842-76a3cb811670
x-user-payload: <base64-user-json>
x-user-timestamp: 1787025600123
x-user-signature: <sha256-hmac-hex>
```

Công thức:

```text
payloadToSign = timestamp + "." + requestId + "." + base64UserPayload
signature = HMAC_SHA256(CANTEEN_INTERNAL_SECRET, payloadToSign)
```

Base64 chỉ giúp truyền JSON qua header; bất kỳ ai cũng có thể tự encode Base64.
HMAC mới là phần giúp Canteen phát hiện payload bị sửa hoặc giả mạo.

### Bước 5: Canteen nhận Request ID

File:

```text
backend/canteen/src/common/middleware/request-id.middleware.ts
```

Canteen nhận lại ID do Gateway chuyển xuống và lưu:

```ts
request.requestContext = {
  requestId,
  startedAt: process.hrtime.bigint(),
};
```

`startedAt` dùng đồng hồ monotonic `hrtime` để đo duration ổn định, không phụ
thuộc việc giờ hệ thống thay đổi.

### Bước 6: Canteen xác minh Gateway và role

Các file:

```text
backend/canteen/src/common/guards/roles.guard.ts
backend/canteen/src/common/security/gateway-signature.service.ts
```

Thứ tự kiểm tra:

1. Lấy `x-user-payload`.
2. Kiểm tra đủ request ID, timestamp và signature.
3. Kiểm tra timestamp còn trong thời hạn cho phép.
4. Tự tính lại HMAC bằng secret của Canteen.
5. So sánh signature bằng `timingSafeEqual`.
6. Decode Base64 và parse JSON.
7. Chỉ nhận user có `_id` hoặc `id` hợp lệ.
8. Đọc `@Roles(...)` hoặc `@Authenticated()`.
9. Cho đi tiếp hoặc ném `401/403`.

Hai decorator có ý nghĩa khác nhau:

```ts
@Authenticated()
```

Chỉ yêu cầu user đã đăng nhập, không giới hạn role cụ thể.

```ts
@Roles(Role.ADMIN, Role.MANAGER, Role.WAITER)
```

Yêu cầu đăng nhập và role phải thuộc danh sách.

### Bước 7: Interceptor bao quanh quá trình xử lý

File:

```text
backend/canteen/src/common/interceptors/http-logging.interceptor.ts
```

Interceptor gọi `next.handle()` để cho pipe, controller và service chạy. Khi
Observable hoàn thành thành công, `tap()` ghi log gồm request ID, user ID,
method, path, status code và tổng thời gian xử lý.

Interceptor không tự xử lý lỗi. Nếu chuỗi xử lý ném exception, exception được
chuyển tới `GlobalExceptionFilter`.

### Bước 8: Pipes kiểm tra input

Global pipe được đăng ký tại `src/main.ts`:

```ts
new ValidationPipe({
  transform: true,
  whitelist: true,
  forbidNonWhitelisted: true,
});
```

Ý nghĩa:

- `transform`: chuyển plain object sang DTO phù hợp;
- `whitelist`: chỉ chấp nhận field đã khai báo trong DTO;
- `forbidNonWhitelisted`: có field lạ thì trả `400`, không âm thầm bỏ qua.

ID bàn đi qua `ParseObjectIdPipe`. Body đi qua `UpdateTableStatusDto`:

```ts
@IsString()
@IsIn(['empty', 'occupied', 'reserved'])
status!: 'empty' | 'occupied' | 'reserved';
```

Các input lỗi:

```http
PATCH /api/canteen/tables/abc/status
```

`abc` không phải MongoDB ObjectId → `400`.

```json
{ "status": "cleaning" }
```

`cleaning` không thuộc danh sách → `400`.

```json
{ "status": "occupied", "isAdmin": true }
```

`isAdmin` không được khai báo trong DTO → `400`.

### Bước 9: Controller gọi service

Controller chỉ nối HTTP layer với business layer:

```ts
async updateTableStatus(
  @Param('id', new ParseObjectIdPipe('ID bàn ăn')) id: string,
  @Body() updateStatusDto: UpdateTableStatusDto,
) {
  return this.tableService.updateTableStatus(id, updateStatusDto);
}
```

Controller không tự truy vấn MongoDB và không tự format lỗi.

### Bước 10: Service xử lý nghiệp vụ

`TableService` thực hiện:

```text
Tìm bàn theo ID
  ├─ Không thấy → throw NotFoundException
  └─ Tìm thấy
       ├─ Cập nhật status
       └─ Lưu MongoDB
```

Không phải endpoint nào cũng gọi cả MongoDB, Redis và RabbitMQ:

- cập nhật trạng thái bàn chủ yếu gọi MongoDB;
- undo/redo menu sử dụng Redis;
- xác nhận đơn hoặc nghiệp vụ kho có thể phát RabbitMQ event.

### Bước 11: Response thành công

Nest serialize document thành JSON và response có header:

```http
HTTP/1.1 200 OK
x-request-id: 8ee70f08-7f49-4187-b842-76a3cb811670
Content-Type: application/json
```

Ví dụ body:

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "name": "Bàn 01",
  "capacity": 4,
  "status": "occupied",
  "updatedAt": "2026-08-18T10:00:00.000Z"
}
```

Hiện chưa có response interceptor bọc mọi response vào `{ success, data }`.
Đó là lựa chọn API contract, không phải điều kiện bắt buộc của production.

---

## 5. Luồng exception

File:

```text
backend/canteen/src/common/filters/global-exception.filter.ts
```

Filter được đăng ký bằng `APP_FILTER`, vì vậy các exception không được xử lý từ
guard, pipe, controller hoặc service sẽ đi tới filter.

```text
Guard/Pipe/Controller/Service ném exception
  ↓
GlobalExceptionFilter
  ├─ Nếu là HttpException: giữ status và message nghiệp vụ
  ├─ Nếu là lỗi không biết: chuyển thành 500
  ├─ Ghi structured log
  └─ Trả body có requestId
```

Ví dụ ObjectId hợp lệ nhưng bàn không tồn tại:

```json
{
  "message": "Bàn ăn với ID '507f1f77bcf86cd799439011' không tồn tại",
  "error": "Not Found",
  "statusCode": 404,
  "requestId": "8ee70f08-7f49-4187-b842-76a3cb811670"
}
```

Ví dụ lỗi không dự kiến:

```json
{
  "statusCode": 500,
  "message": "Internal server error",
  "requestId": "8ee70f08-7f49-4187-b842-76a3cb811670"
}
```

Stack trace chỉ ghi ở backend log, không trả cho client.

### Bảng điểm dừng

| Trường hợp             | Nơi dừng              | Status thường gặp | Controller chạy? |
| ---------------------- | --------------------- | ----------------: | ---------------: |
| Gọi quá nhiều          | Gateway rate limit    |               429 |            Không |
| JWT thiếu/hết hạn      | Gateway JWT guard     |               401 |            Không |
| Sai role tại Gateway   | Gateway roles guard   |               403 |            Không |
| HMAC sai/hết hạn       | Canteen roles guard   |               401 |            Không |
| Sai ObjectId           | ParseObjectIdPipe     |               400 |            Không |
| Body sai DTO           | ValidationPipe        |               400 |            Không |
| Không tìm thấy dữ liệu | Domain service        |               404 |               Có |
| Xung đột nghiệp vụ     | Domain service        |               409 |               Có |
| Lỗi không dự kiến      | Bất kỳ phần xử lý nào |               500 |       Tùy vị trí |

---

## 6. Request ID và cách debug thực tế

Ví dụ frontend nhận lỗi:

```json
{
  "statusCode": 500,
  "message": "Internal server error",
  "requestId": "8ee70f08-7f49-4187-b842-76a3cb811670"
}
```

Developer tìm trong hệ thống log:

```text
requestId="8ee70f08-7f49-4187-b842-76a3cb811670"
```

Structured log thành công có dạng:

```json
{
  "timestamp": "2026-08-18T10:00:00.123Z",
  "service": "canteen",
  "event": "http_request_completed",
  "requestId": "8ee70f08-7f49-4187-b842-76a3cb811670",
  "userId": "64b7f0f0f0f0f0f0f0f0f0f0",
  "method": "PATCH",
  "path": "/api/canteen/tables/507f1f77bcf86cd799439011/status",
  "statusCode": 200,
  "durationMs": 42.7
}
```

Log lỗi có event:

```text
http_request_rejected  # thường là lỗi 4xx
http_request_failed    # lỗi 5xx
```

Không ghi password, JWT, OTP, raw authorization header hoặc secret vào log.

---

## 7. Monitoring, Telegram và Discord

Telegram/Discord không phải tầng thứ chín trong Nest request lifecycle.

Luồng đề xuất:

```text
Canteen StructuredLogger
  ↓ stdout/stderr JSON
Log collector (Promtail/Filebeat/Agent)
  ↓
Loki / ELK / Datadog / Cloud Logging
  ↓
Alert rule
  ├─ error rate tăng cao
  ├─ nhiều response 500
  ├─ service readiness fail
  └─ latency vượt ngưỡng
       ↓
Telegram / Discord / Email / PagerDuty
```

Không nên gọi webhook Telegram/Discord trực tiếp trong exception filter vì:

- làm request lỗi chậm hơn;
- webhook hỏng có thể tạo lỗi phụ;
- sự cố hàng loạt gây spam;
- khó rate limit và gom nhóm cảnh báo.

---

## 8. Health check cho deploy

Các endpoint:

```http
GET /health/live
GET /health
GET /health/ready
```

### Liveness

`GET /health/live` chỉ trả lời process Nest còn sống hay không. Endpoint này
không nên thất bại chỉ vì MongoDB tạm thời mất kết nối, tránh việc orchestrator
restart service liên tục.

### Readiness

`GET /health` và `GET /health/ready` kiểm tra:

- MongoDB connection state;
- Redis client ready;
- RabbitMQ connection/channel ready.

Nếu một dependency chưa sẵn sàng, response là `503`. Load balancer hoặc
Kubernetes có thể tạm ngừng chuyển traffic đến instance đó.

Ví dụ thành công:

```json
{
  "status": "ok",
  "service": "canteen",
  "dependencies": {
    "mongodb": "up",
    "redis": "up",
    "rabbitmq": "up"
  },
  "timestamp": "2026-08-18T10:00:00.000Z"
}
```

---

## 9. Cấu hình môi trường

Gateway và Canteen phải dùng cùng secret:

```dotenv
# backend/gateway/.env
NODE_ENV=production
CANTEEN_INTERNAL_SECRET=replace_with_a_random_secret_at_least_32_characters
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=120
```

```dotenv
# backend/canteen/.env
NODE_ENV=production
CANTEEN_INTERNAL_SECRET=replace_with_a_random_secret_at_least_32_characters
CANTEEN_SIGNATURE_MAX_AGE_MS=300000
```

Quy tắc:

- không commit `.env` thật;
- không dùng secret mẫu ở production;
- secret tối thiểu 32 ký tự;
- hai service phải có cùng giá trị;
- khi rotate secret cần triển khai hai service có phối hợp;
- production luôn bắt buộc chữ ký;
- local có thể đặt `CANTEEN_REQUIRE_SIGNATURE=false` trong giai đoạn chuyển đổi.

Tạo secret bằng công cụ quản lý secret của môi trường deploy. Không gửi secret
qua chat, log hoặc commit Git.

---

## 10. Chạy kiểm tra

### Gateway

```bash
cd backend/gateway
npm run build
```

### Canteen

```bash
cd backend/canteen
npm run build
npm test -- --runInBand
```

Unit test HMAC kiểm tra ba trường hợp:

- chữ ký hợp lệ được chấp nhận;
- chữ ký sai bị từ chối;
- payload quá hạn bị từ chối.

### Gọi thử endpoint qua Gateway

```bash
curl -i -X PATCH \
  http://localhost:3000/api/canteen/tables/507f1f77bcf86cd799439011/status \
  -H 'Authorization: Bearer <access-token>' \
  -H 'Content-Type: application/json' \
  -d '{"status":"occupied"}'
```

Không cần tự tạo `x-user-payload` hoặc HMAC khi gọi qua Gateway. Gateway thực
hiện việc đó sau khi xác thực JWT.

---

## 11. Khi thêm endpoint mới

Ví dụ thêm endpoint hủy order:

```http
PATCH /api/canteen/orders/:id/cancel
```

Checklist:

1. Tạo hoặc cập nhật DTO trong `modules/order/dto`.
2. Thêm decorator validation cho mọi input từ client.
3. Thêm route trong `order.controller.ts`.
4. Chọn `@Authenticated()` hoặc `@Roles(...)`.
5. Dùng `ParseObjectIdPipe` cho MongoDB ID.
6. Đặt business logic trong `order.service.ts`.
7. Ném exception Nest phù hợp: `NotFoundException`, `ConflictException`...
8. Nếu phát event, gọi RabbitMQ adapter từ service.
9. Bổ sung route tương ứng tại Gateway controller/service.
10. Viết unit test cho nghiệp vụ và e2e test cho route quan trọng.

Không cần tự gọi logger thành công ở mỗi controller; global interceptor đã ghi
access log. Chỉ thêm domain log khi có sự kiện nghiệp vụ quan trọng mà access
log không thể diễn đạt.

---

## 12. Giới hạn hiện tại và hướng nâng cấp

### Rate limit đang lưu trong memory

Mỗi Gateway instance có một `Map` riêng. Khi deploy nhiều replica, tổng quota
không còn chính xác. Cần chuyển sang một trong các lựa chọn:

- Redis-backed rate limiter;
- rate limit tại Nginx/Ingress;
- API management như Kong, Traefik hoặc cloud gateway.

### Chưa có central monitoring trong repository

Service đã xuất structured log nhưng Loki/ELK/Datadog và alert rule vẫn là hạ
tầng deploy riêng. Telegram/Discord chưa được gọi trực tiếp trong code.

### Chưa chuẩn hóa success response toàn cục

Response hiện giữ nguyên dữ liệu controller trả về. Chỉ thêm response
interceptor nếu frontend và API contract thống nhất một envelope cụ thể.

### Canteen phải nằm sau Gateway

Production nên giới hạn network để client bên ngoài không gọi trực tiếp port
Canteen. HMAC bảo vệ user payload nhưng private network/service policy vẫn là
một lớp phòng thủ quan trọng.

---

## 13. Tóm tắt dễ nhớ

```text
Request ID  = request nào?
JWT Guard   = token có thật và còn hiệu lực không?
Roles Guard = user có quyền không?
HMAC        = payload có thật sự đến từ Gateway không?
Pipe        = input có hợp lệ không?
Controller  = endpoint gọi use case nào?
Service     = nghiệp vụ xử lý ra sao?
Interceptor = request thành công mất bao lâu?
Filter      = lỗi được log và trả về thế nào?
Health      = instance có sống và sẵn sàng nhận traffic không?
Monitoring  = toàn hệ thống có dấu hiệu bất thường không?
```

Luồng này tạo nền móng production tốt, nhưng production-ready còn phụ thuộc vào
network policy, secret management, test coverage, backup, metrics, dashboard,
deployment strategy và quy trình vận hành.
