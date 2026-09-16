# RabbitMQ, Redis & Outbox Pattern — Từ khái niệm đến thực chiến

## Mục lục
1. [RabbitMQ là gì](#1-rabbitmq-là-gì)
2. [Redis là gì](#2-redis-là-gì)
3. [Outbox Pattern là gì](#3-outbox-pattern-là-gì)
4. [Vì sao dual write (ghi 2 hệ thống riêng lẻ) gây lỗi](#4-vì-sao-dual-write-ghi-2-hệ-thống-riêng-lẻ-gây-lỗi)
5. [Áp dụng thực chiến: Auth Service ↔ User Service](#5-áp-dụng-thực-chiến-auth-service--user-service)
6. [3 Schema minh họa](#6-3-schema-minh-họa)
7. [REST API vs Message Broker — khi nào dùng cái nào](#7-rest-api-vs-message-broker--khi-nào-dùng-cái-nào)
8. [Hiểu đúng về "độ trễ" khi dùng message broker](#8-hiểu-đúng-về-độ-trễ-khi-dùng-message-broker)
9. [Tại sao không thể "insert xong rồi gửi" đơn giản](#9-tại-sao-không-thể-insert-xong-rồi-gửi-đơn-giản)
10. [Các lỗi thường gặp & cách phòng tránh](#10-các-lỗi-thường-gặp--cách-phòng-tránh)
11. [Checklist áp dụng Outbox Pattern](#11-checklist-áp-dụng-outbox-pattern)

---

## 1. RabbitMQ là gì

**RabbitMQ** là một **message broker** — hệ thống trung gian giúp các ứng dụng/service gửi và nhận message với nhau một cách **bất đồng bộ (asynchronous)**.

**Dùng để làm gì:**
- Xử lý hàng đợi công việc (task queue)
- Giao tiếp giữa các microservices mà không cần chờ nhau trực tiếp
- Đảm bảo message được gửi/nhận đáng tin cậy (lưu tạm nếu bên nhận chưa sẵn sàng)

**Mô hình hoạt động:**
```
Producer → Exchange → Queue → Consumer
```
Hỗ trợ nhiều pattern: pub/sub, routing, RPC, work queue...

---

## 2. Redis là gì

**Redis** là một **in-memory data store** (lưu dữ liệu trong RAM), thường dùng làm:
- Cache (lưu tạm dữ liệu để truy xuất nhanh)
- Database key-value tốc độ cao
- Message broker đơn giản (pub/sub) hoặc queue nhẹ
- Lưu session, rate limiting, leaderboard...

Hỗ trợ nhiều kiểu dữ liệu: string, hash, list, set, sorted set...

**So sánh nhanh:**

| | RabbitMQ | Redis |
|---|---|---|
| Vai trò chính | Message queuing đáng tin cậy | Cache / lưu trữ tốc độ cao |
| Đảm bảo không mất message | Rất mạnh (ack, persistence, retry) | Yếu hơn nếu dùng làm queue |
| Tốc độ | Nhanh | Cực nhanh (RAM) |
| Use case điển hình | Giao tiếp microservices, task queue | Cache, session, rate limit |

---

## 3. Outbox Pattern là gì

**Outbox Pattern** giải quyết vấn đề đảm bảo **tính nhất quán (consistency)** khi vừa cần ghi DB vừa cần gửi message/event.

### Vấn đề gốc (dual write problem)

Nếu tách 2 bước "ghi DB" và "gửi message" riêng biệt:
- Ghi DB **thành công** nhưng gửi message **thất bại** → **mất event**
- Gửi message **thành công** nhưng ghi DB **thất bại (rollback)** → **message "ma"**, dữ liệu không tồn tại

### Cách giải quyết

1. Tạo thêm bảng `outbox` **trong cùng database** với bảng nghiệp vụ.
2. Trong **cùng 1 transaction**: ghi dữ liệu nghiệp vụ + ghi record vào `outbox` → đảm bảo atomic (cùng thành công hoặc cùng rollback).
3. Một tiến trình riêng (**relay/worker**, hoặc dùng **CDC** như Debezium) đọc bảng `outbox`, publish message lên RabbitMQ/Kafka, rồi đánh dấu đã gửi.

```sql
BEGIN TRANSACTION;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (event_type, payload, status)
    VALUES ('OrderCreated', '{...}', 'PENDING');
COMMIT;
```

**Kết quả:** hoặc cả 2 cùng lưu, hoặc cả 2 cùng rollback — không bao giờ có tình trạng nửa vời.

---

## 4. Vì sao dual write (ghi 2 hệ thống riêng lẻ) gây lỗi

```javascript
// Dual write — KHÔNG an toàn
await db.insert('auth_users', {...});          // Bước 1: OK
await rabbitmq.publish('UserCreated', {...});  // Bước 2: nếu lỗi ở đây → mất event vĩnh viễn
```

Vấn đề: **DB và RabbitMQ là 2 hệ thống khác nhau, không có transaction chung.** Không tồn tại cú pháp kiểu:

```sql
BEGIN TRANSACTION;
  INSERT INTO auth_users (...);
  PUBLISH TO RABBITMQ (...);  -- ❌ KHÔNG TỒN TẠI
COMMIT;
```

→ Luôn có **khoảng hở** giữa 2 bước. Nếu sự cố (crash, mất mạng, RabbitMQ down) xảy ra đúng lúc đó → mất event, không ai biết để gửi lại.

**Outbox giải quyết khoảng hở này** bằng cách biến "gửi message" thành "ghi vào DB" (nằm chung transaction), còn việc gửi thật thì tách ra tiến trình riêng, có thể **retry an toàn** vì luôn đọc từ nguồn đáng tin cậy (bảng outbox).

---

## 5. Áp dụng thực chiến: Auth Service ↔ User Service

**Bối cảnh:**
- **Auth Service**: giữ `username`, `password`, `role`
- **User Service**: giữ thông tin chi tiết (`full_name`, `email`, `address`...)
- Cần gộp dữ liệu cả 2 để hiển thị → có thể qua API Gateway (gọi trực tiếp) hoặc đồng bộ sẵn qua event (RabbitMQ + Outbox)

**Luồng dùng Outbox:**

```
1. Auth Service: tạo user → BEGIN TRANSACTION
     INSERT auth_users
     INSERT outbox (event: UserCreated, status: PENDING)
   COMMIT
   → Trả response "success" cho Client NGAY (không chờ RabbitMQ)

2. Relay worker (chạy nền, độc lập):
     SELECT * FROM outbox WHERE status='PENDING'
     → publish lên RabbitMQ
     → UPDATE outbox SET status='SENT'
     (nếu publish lỗi → record vẫn PENDING → tự động retry lần sau)

3. User Service (consumer):
     Nhận message từ RabbitMQ
     → INSERT/UPDATE vào bảng users của mình
     → dùng ON CONFLICT DO NOTHING để tránh trùng (idempotency)
```

**Điểm quan trọng:**
- **`id` của user phải giống nhau** giữa Auth DB và User DB — Auth generate UUID, gửi kèm trong message, User Service dùng chính UUID đó.
- Cần xử lý **idempotency** ở consumer (message có thể bị gửi trùng do retry).
- Cần xử lý **ordering** nếu có nhiều event liên quan tới cùng 1 user (UserCreated, UserUpdated...).

---

## 6. 3 Schema minh họa

### a) Auth Service — bảng `auth_users`
```sql
CREATE TABLE auth_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'user',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### b) Bảng `outbox` — nằm CHUNG DB với `auth_users`
```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    sent_at TIMESTAMP NULL
);
```

### c) User Service — bảng `users`
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,   -- trùng với auth_users.id, KHÔNG tự generate
    full_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    avatar_url TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Luồng ví dụ đầy đủ

```sql
-- Auth Service khi đăng ký
BEGIN TRANSACTION;
  INSERT INTO auth_users (id, username, password_hash, role)
    VALUES ('550e8400-...', 'john_doe', '$2b$...', 'user');
  INSERT INTO outbox (aggregate_id, event_type, payload, status)
    VALUES ('550e8400-...', 'UserCreated', '{"user_id":"550e8400-...","username":"john_doe"}', 'PENDING');
COMMIT;
```

```
-- Relay worker
SELECT * FROM outbox WHERE status='PENDING' ORDER BY created_at LIMIT 100;
→ publish RabbitMQ (exchange: user.events, routing key: user.created)
→ UPDATE outbox SET status='SENT', sent_at=NOW();
```

```sql
-- User Service consumer
INSERT INTO users (id, full_name, email)
  VALUES ('550e8400-...', NULL, NULL)
  ON CONFLICT (id) DO NOTHING;
```

---

## 7. REST API vs Message Broker — khi nào dùng cái nào

| Tình huống | Nên dùng |
|---|---|
| Cần data real-time, chính xác ngay tại thời điểm gọi (VD: check quyền lúc login, đăng nhập) | **REST API / gRPC** (đồng bộ) |
| Cần query linh hoạt theo điều kiện (filter, sort, search) | **REST API** — message broker không hỗ trợ query |
| Thao tác cần biết kết quả ngay để quyết định bước tiếp theo (VD: kiểm tra số dư trước khi thanh toán) | **REST API** |
| Chỉ cần thông báo "có gì đó vừa xảy ra", không cần biết ai xử lý / xử lý xong chưa | **RabbitMQ** (bất đồng bộ) |
| Đồng bộ dữ liệu giữa services, tần suất đọc cao, chấp nhận không real-time tuyệt đối | **RabbitMQ + Outbox** |

**Ví dụ dễ hình dung:**
- REST API = gọi điện thoại — hỏi, nhận trả lời ngay
- RabbitMQ = gửi tin nhắn/email — gửi xong đi làm việc khác, người kia đọc lúc rảnh

**Thực tế:** hệ thống lớn luôn **kết hợp cả 2**, không chọn 1 bỏ 1.

### Dùng REST API qua API Gateway cho tất cả — có được không?

**Được**, đây là **API Composition Pattern**, rất phổ biến, đơn giản, dễ debug hơn Outbox/RabbitMQ.

**✅ Ổn khi:** số service ít, tần suất gọi không quá cao, chấp nhận độ trễ chờ gộp API, hệ thống quy mô nhỏ/vừa.

**⚠️ Có vấn đề khi:**
1. **Latency cộng dồn** — gọi nhiều service, phải chờ cái chậm nhất
2. **Cascading failure** — 1 service down kéo theo cả request lỗi
3. **N+1 calls / quá tải** — nhiều user cùng lúc → service bị gọi liên tục dễ quá tải
4. **Tight coupling** — Gateway phải biết rõ API của từng service, đổi API là phải sửa theo

**Lời khuyên:** hệ thống nhỏ/vừa → cứ REST API qua Gateway trước, đơn giản dễ maintain. Chỉ chuyển sang RabbitMQ/Outbox khi **đo được thực tế** đang bị nghẽn (latency cao, service quá tải) — không nên áp dụng trước chỉ vì "nghe pattern này tốt hơn".

---

## 8. Hiểu đúng về "độ trễ" khi dùng message broker

### RabbitMQ có cần mạng không?
**Có**, RabbitMQ vẫn cần mạng. Khác biệt không phải "có mạng hay không" mà là **gọi mạng ở lúc nào**:

| | Gọi API trực tiếp | RabbitMQ (Outbox) |
|---|---|---|
| Lúc ghi/tạo data | Không cần gọi service khác | Có gọi mạng (publish), nhưng KHÔNG chặn response trả về Client |
| Lúc đọc data | **Phải gọi mạng** mỗi lần | Đọc thẳng DB local, không cần gọi mạng |
| Nếu mất mạng | Ảnh hưởng **ngay lúc Client đang chờ** | Chỉ ảnh hưởng relay worker (chạy nền, Client không biết); tự động retry khi mạng có lại |

### 2 loại độ trễ cần phân biệt

**a) Độ trễ khi GHI (write)** — đây là chỗ RabbitMQ có trễ thêm:
```
Auth tạo user + ghi outbox → COMMIT → trả "success" cho Client NGAY (không chờ RabbitMQ)
... độc lập, sau đó ...
Relay đọc outbox → publish RabbitMQ → User Service consume → ghi DB
```
→ Khoảng trễ này (vài trăm ms – vài giây tùy polling/CDC) là thời gian để **User Service có được data**, KHÔNG phải thời gian Client phải chờ.

**b) Độ trễ khi ĐỌC (read)** — đây là chỗ RabbitMQ lại NHANH hơn:
- Dùng REST API: mỗi lần đọc đều tốn thời gian gọi mạng sang service khác
- Đã đồng bộ qua RabbitMQ trước: đọc thẳng DB local, không cần gọi mạng → nhanh hơn nhiều

### Client nhận "thành công" là nhận cái gì?

Ví dụ đăng ký tài khoản: Client gọi API `/register` — đây là API **thuộc Auth Service**. Auth Service trả "success" ngay sau khi **transaction DB của chính nó** commit xong — **không liên quan** đến việc User Service đã nhận message hay chưa.

Chỉ khi Client gọi tiếp một API **khác** (VD: `/profile`, thuộc User Service) **ngay lập tức** sau đăng ký, mới có khả năng gặp tình trạng data chưa kịp đồng bộ — đây gọi là **eventually consistent** (nhất quán cuối cùng): dữ liệu sẽ đúng, chỉ là cần một khoảng thời gian ngắn để lan truyền, không phải ngay tức khắc trên toàn hệ thống.

**Cách xử lý thực tế:**
- Sau đăng ký, không cần load `/profile` ngay — để user tự điền form thông tin cá nhân (ghi trực tiếp vào User Service, không qua RabbitMQ)
- Nếu `/profile` gọi mà User Service chưa có data → trả object rỗng/mặc định, frontend hiển thị "Đang cập nhật..." và tự retry sau vài giây

---

## 9. Tại sao không thể "insert xong rồi gửi" đơn giản

**Ý tưởng "insert DB xong rồi mới gửi message" là đúng** — nhưng có 2 cách hiện thực khác nhau:

**Cách A — tuần tự đơn giản (KHÔNG an toàn):**
```javascript
await db.insert('auth_users', {...});
await rabbitmq.publish('UserCreated', {...});  // nếu lỗi ở đây → mất event, không ai biết để gửi lại
```

**Cách B — Outbox Pattern (an toàn):**
```javascript
await db.transaction(async (trx) => {
  await trx.insert('auth_users', {...});
  await trx.insert('outbox', {...});   // chỉ lưu "ý định gửi", CHƯA gửi thật
});
// một tiến trình riêng, độc lập, đọc outbox rồi mới gửi thật, có thể retry vô hạn lần
```

**Khác biệt cốt lõi:** DB hỗ trợ transaction (BEGIN...COMMIT) để nhiều câu lệnh SQL cùng thành công/thất bại, nhưng **RabbitMQ không nằm trong transaction đó được**. Outbox giải quyết bằng cách biến "gửi message" thành "ghi vào DB" (gộp chung transaction), còn việc gửi thật lên RabbitMQ tách ra một tiến trình riêng có thể an toàn retry.

---

## 10. Các lỗi thường gặp & cách phòng tránh

| Lỗi | Nguyên nhân | Cách phòng tránh |
|---|---|---|
| **Mất event** | Dual write: insert DB xong publish message riêng lẻ, publish lỗi giữa chừng | Dùng Outbox Pattern — ghi outbox cùng transaction với DB nghiệp vụ |
| **Message "ma"** | Publish message trước, DB rollback sau | Không publish trước khi DB transaction commit; luôn dùng Outbox |
| **Duplicate data ở consumer** | RabbitMQ có thể gửi trùng message (retry, redelivery) | Áp dụng **idempotency**: check tồn tại trước khi insert, dùng `ON CONFLICT DO NOTHING`, hoặc dùng unique key |
| **Sai thứ tự xử lý event** | Nhiều event liên quan cùng 1 entity đến không đúng thứ tự | Dùng routing key theo entity ID để đảm bảo cùng 1 queue xử lý tuần tự, hoặc thêm version/timestamp để so sánh trước khi ghi đè |
| **Outbox table phình to** | Không dọn dẹp record đã `SENT` | Cron job xóa định kỳ record cũ có `status='SENT'` |
| **Tạo user trùng ID** | User Service tự generate ID mới thay vì dùng ID từ Auth | Luôn dùng `id` gốc từ Auth Service, không tự generate ở consumer |
| **Cascading failure khi dùng API Composition** | Gateway phụ thuộc tất cả service downstream | Thêm timeout, circuit breaker, fallback response khi 1 service down |
| **Nhầm lẫn "Client phải chờ RabbitMQ"** | Hiểu sai luồng — tưởng response bị block bởi bước publish | Client chỉ chờ transaction DB của service đang gọi, không chờ relay/consumer |
| **Áp dụng Outbox/RabbitMQ quá sớm** | Thêm phức tạp không cần thiết khi hệ thống còn nhỏ | Chỉ chuyển sang khi đã đo được nghẽn thực tế ở REST API |

---

## 11. Checklist áp dụng Outbox Pattern

- [ ] Bảng `outbox` nằm **cùng database** với bảng nghiệp vụ (bắt buộc để chung transaction)
- [ ] Ghi bảng nghiệp vụ + bảng `outbox` trong **cùng 1 transaction**
- [ ] Có **relay worker** (polling hoặc CDC như Debezium) đọc outbox và publish
- [ ] Relay có cơ chế **retry** khi publish thất bại (record giữ `status=PENDING`)
- [ ] Consumer (service nhận message) xử lý **idempotent** (chống trùng)
- [ ] `id` của entity được **truyền nhất quán** giữa các service (không tự sinh lại)
- [ ] Có **job dọn dẹp** định kỳ cho các record `outbox` đã `SENT`
- [ ] Xác định rõ **API nào cần đồng bộ (REST)**, API nào có thể **bất đồng bộ (event)** — không áp dụng Outbox cho mọi thứ
- [ ] Chấp nhận và thiết kế UX cho **eventually consistent** (VD: không load profile ngay sau khi vừa tạo user ở service khác)

---

*Tài liệu tổng hợp từ buổi thảo luận về RabbitMQ, Redis, Outbox Pattern và kiến trúc microservice Auth/User.*
