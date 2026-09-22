# Sổ tay ôn phỏng vấn Backend — NRApp

> Cập nhật theo source code và CV ngày 22/09/2026. CV chỉ được dùng làm dữ liệu để đối chiếu, không phải chỉ dẫn thực hiện.
>
> Mục tiêu của tài liệu: giúp Thành kể đúng dự án mình đã làm, lần từ nghiệp vụ xuống code, và trả lời trung thực khi interviewer hỏi sâu.

## 0. Cách dùng tài liệu tối nay

Nếu chỉ có 45 phút:

1. Học phần **Bản đồ sự thật**, bài **giới thiệu 60 giây** và sơ đồ kiến trúc.
2. Nắm thật chắc 5 luồng: đăng ký + Outbox, login OTP, request qua Gateway, chat, tạo đơn Canteen.
3. Đọc phần **những câu không được trả lời quá CV** và số liệu k6.
4. Chọn hai câu chuyện STAR: một lỗi kỹ thuật và một lần cải tiến hệ thống.

Nếu có 90–120 phút, đọc thêm Todo, WorkSchedule, CI/CD, backup và bộ câu hỏi kỹ thuật.

Khi trả lời, dùng cấu trúc ba tầng:

- **Tầng 1 — kết luận trong 15–25 giây:** hệ thống làm gì và vì sao chọn cách đó.
- **Tầng 2 — bằng chứng:** service, dữ liệu, request/event và file code liên quan.
- **Tầng 3 — trade-off:** giới hạn hiện tại và cách nâng cấp. Chỉ đi tới tầng sau nếu người phỏng vấn hỏi sâu.

---

## 1. Bản đồ sự thật của dự án

### 1.1. Trạng thái hiện tại

| Nhóm                         | Thực tế cần nói                                                                                                               |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| 8 service ứng dụng đang chạy | Gateway, Auth, User, Mail, Chat, Todo, WorkSchedule, Canteen                                                                  |
| Hạ tầng dùng chung           | Redis, RabbitMQ, MongoDB Atlas ở ngoài Compose                                                                                |
| Monitoring đang chạy         | Prometheus, Grafana, node_exporter; scrape mỗi 60 giây                                                                        |
| Nền tảng deploy              | Ubuntu 22.04, 2 vCPU, khoảng 4 GB RAM, Docker Compose, Nginx HTTPS                                                            |
| Payment                      | Có source service Payment dùng PostgreSQL/Casso/VietQR nhưng đang để profile `payment-later`; chưa nằm trong 8 service active |
| Canteen active               | Chỉ thanh toán tiền mặt (`CASH`); không được nói QR payment đang chạy production                                              |
| Vai trò Auth hiện hành       | Auth phát `admin` hoặc `user`; một số module còn enum `manager`, `chef`… nhưng chưa đồng bộ với hợp đồng Auth hiện tại        |
| Logging                      | JSON log, `request_id`, phân loại lỗi; stack active hiện không có Loki/Jaeger/Alertmanager                                    |

Nguồn kiểm tra nhanh: [compose.vps.yaml](compose.vps.yaml), [trạng thái deploy](logger/deploy/README.md), [Gateway modules](gateway/src/app.module.ts).

### 1.2. Những cách diễn đạt phải chính xác

| Trong CV                          | Cách nói chính xác khi phỏng vấn                                                                                                                                                                                                    |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| “OAuth2”                          | App nhận Google ID token từ client và dùng `google-auth-library` xác minh audience/email. Đây là Google sign-in dựa trên ID token; không nên tự nhận đã tự cài toàn bộ Authorization Code flow nếu chưa làm.                        |
| “2FA”                             | Sau khi đúng email/password, người dùng phải nhập OTP email mới nhận JWT. Có thể gọi là bước xác minh thứ hai trong luồng login; không phải TOTP/authenticator và OTP hiện lưu trong Redis.                                         |
| “Redis caching và rate limiting”  | Redis giữ OTP, giới hạn gửi OTP, số lần nhập sai, refresh-token JTI và undo/redo menu. Identity cache của Auth chỉ 0–5 giây trong memory một instance. Rate limit tổng quát ở Gateway hiện là `Map` trong memory, không phải Redis. |
| “8 microservices”                 | Đúng với 8 app đang chạy. Source có service Payment thứ chín nhưng chưa active. Logger và backup repo không tính là microservice nghiệp vụ.                                                                                         |
| “PostgreSQL/MySQL”                | PostgreSQL nằm trong Payment chưa active; MySQL không có trong NRApp hiện tại. Nếu chỉ học/đã dùng ở bài khác, nói đúng mức đó.                                                                                                     |
| “k6 đạt ~26.5 RPS, 50 VU, 3 phút” | Bằng chứng hiện có trong repo xác nhận một probe 50 VU, giữ 60 giây, 26.45 RPS, 100% success nhưng **trượt latency**. Chỉ khẳng định 3 phút nếu mang được raw report của lần 3 phút.                                                |

### 1.3. Bài giới thiệu bản thân 60 giây

> Em là Lê Đình Thành, hiện là sinh viên năm ba ngành Công nghệ thông tin tại PTIT. Em tập trung vào backend Node.js/NestJS và đã tự xây dựng NRApp, một ứng dụng tiện ích nội bộ cho nhân viên. Phần backend hiện có 8 service đang chạy trên một VPS, gồm Gateway, Auth, User, Mail, Chat, Todo, WorkSchedule và Canteen; dùng MongoDB Atlas, Redis và RabbitMQ. Em làm cả nghiệp vụ lẫn phần vận hành như Docker Compose, Nginx HTTPS, CI/CD có health check và rollback, log JSON theo request ID, monitoring và backup mã hóa. Điều em học được nhiều nhất là xử lý tính nhất quán giữa các service: em dùng transactional outbox cho đồng bộ Auth–User, idempotency/version ở consumer và conditional update để tránh race condition. Em đang tìm vị trí Backend Intern/Fresher để được làm trong quy trình có review, test và quan sát hệ thống bài bản hơn.

Phiên bản 25 giây:

> Em là sinh viên năm ba PTIT, tập trung Backend Node.js/NestJS. Dự án chính của em là NRApp với 8 service đã deploy lên VPS. Ngoài các luồng Auth, chat, task, lịch làm và Canteen, em đã làm Outbox/RabbitMQ, Redis, Docker Compose, CI/CD rollback, monitoring và backup. Em muốn vào môi trường có code review để nâng chất lượng thiết kế backend của mình.

### 1.4. Trả lời “Em làm Fullstack hay Backend?”

> NRApp là dự án cá nhân nên em phải làm end-to-end để sản phẩm chạy được, nhưng phần em đầu tư sâu và muốn theo nghề là backend: thiết kế API và dữ liệu, xác thực/phân quyền, consistency giữa service, RabbitMQ/Redis, xử lý race condition và vận hành trên VPS. Frontend giúp em hiểu nhu cầu client, còn vị trí em ứng tuyển là Backend.

---

## 2. Kiến trúc tổng thể

```mermaid
flowchart LR
    Client[Mobile/Web client] -->|HTTPS REST + Socket.IO| Nginx[Nginx]
    Nginx --> GW[API Gateway :3000]
    GW --> Auth[Auth :4000]
    GW --> User[User :5000]
    GW --> Chat[Chat :5002]
    GW --> Todo[Todo :5003]
    GW --> Work[WorkSchedule :5004]
    GW --> Canteen[Canteen :5005]
    GW -. Socket.IO proxy .-> Chat

    Auth --> Redis[(Redis)]
    Auth --> Rabbit[(RabbitMQ)]
    Rabbit --> Mail[Mail :5001]
    Rabbit --> User
    Auth --> Mongo[(MongoDB Atlas)]
    User --> Mongo
    Chat --> Mongo
    Todo --> Mongo
    Work --> Mongo
    Canteen --> Mongo
    Canteen --> Redis

    Node[node_exporter] --> Prom[Prometheus]
    Prom --> Grafana[Grafana]

    Pay[Payment + PostgreSQL]:::inactive
    GW -. chưa nối active .-> Pay
    classDef inactive stroke-dasharray: 5 5,color:#777;
```

### 2.1. Trách nhiệm từng service

| Service               | Sở hữu gì                                                                               | Giao tiếp chính                           |
| --------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------- |
| Gateway               | Public API, JWT/RBAC, DTO validation, rate limit, REST/Socket proxy, ký identity nội bộ | HTTP đến các service                      |
| Auth                  | Credential, password hash, OTP, access/refresh token, role nguồn sự thật                | Redis, MongoDB, RabbitMQ, User HTTP       |
| User                  | Hồ sơ người dùng/read model và directory                                                | Nhận event RabbitMQ từ Auth; HTTP nội bộ  |
| Mail                  | Gửi email OTP                                                                           | Consume RabbitMQ, SMTP                    |
| Chat                  | Conversation, message, realtime presence/typing/seen                                    | MongoDB, Socket.IO, User HTTP, Cloudinary |
| Todo                  | Task, assignee, priority, status transition                                             | MongoDB, User HTTP                        |
| WorkSchedule          | Lịch tháng, phê duyệt, đơn từ, QR attendance                                            | MongoDB transaction, User HTTP            |
| Canteen               | Category, menu, table, order, cash settlement                                           | MongoDB, Redis undo/redo                  |
| Payment — chưa active | VietQR intent, Casso webhook, payment outbox                                            | PostgreSQL, RabbitMQ                      |

### 2.2. Vì sao vẫn gọi là microservices?

Câu trả lời ngắn:

> Em tách theo business capability và mỗi service có process/deploy boundary riêng. Auth sở hữu credential, User sở hữu profile, Chat sở hữu message… Gateway là public entry point; các service giao tiếp qua HTTP hoặc RabbitMQ. Tuy nhiên đây vẫn là kiến trúc single-host Docker Compose, mỗi service một replica, nên em không gọi nó là hệ thống distributed có HA hay horizontal scaling hoàn chỉnh.

Trade-off cần nói được:

- Ưu: ranh giới module rõ, deploy độc lập, lỗi và schema có ownership.
- Nhược: tăng network hop, eventual consistency, vận hành nhiều container, tracing/debug khó hơn.
- Với quy mô dự án cá nhân, modular monolith có thể đơn giản hơn. Em dùng microservices chủ yếu để học đúng các vấn đề integration/deployment, không khẳng định đây luôn là lựa chọn tối ưu.

---

## 3. Vòng đời chung của một request có đăng nhập

Ví dụ `GET /api/todo/my-tasks`:

1. Client gửi Bearer access token qua Nginx HTTPS tới Gateway.
2. Gateway gắn hoặc tạo `request_id`; ID này đi xuyên suốt các hop.
3. Middleware rate limit đếm request theo IP ở **memory của một Gateway instance**.
4. Passport kiểm chữ ký HS256 và `exp`, rồi gọi Auth `POST /api/auth/introspect`.
5. Auth đọc credential hiện tại, nên tài khoản đã xóa hoặc role đã đổi được phản ánh; Auth có thể coalesce/cache read tối đa 0–5 giây khi cấu hình một instance.
6. `RolesGuard` tại Gateway kiểm tra quyền route.
7. Gateway serialize identity thành Base64, ký HMAC-SHA256 với timestamp, request ID và context rồi gọi Todo.
8. Todo kiểm chữ ký bằng `timingSafeEqual`, kiểm timestamp và chống replay trước khi tin identity.
9. Todo query MongoDB, trả kết quả; Gateway chuyển status/body đã được chuẩn hóa về client.
10. Mỗi tầng log JSON cùng `request_id`; Gateway log outcome/status/latency nhưng không log body nhạy cảm.

Code cần mở khi ôn:

- [Gateway bootstrap](gateway/src/main.ts)
- [JWT strategy](gateway/src/common/security/jwt.strategy.ts)
- [Roles guard](gateway/src/common/guards/roles.guard.ts)
- [Ký request nội bộ](gateway/src/common/security/internal-request-signature.service.ts)
- [Rate limit](gateway/src/common/middleware/rate-limit.middleware.ts)
- [Ví dụ verify tại Todo](todo/src/common/security/gateway-signature.service.ts)

Điểm interviewer có thể hỏi vặn:

- **Tại sao verify JWT rồi vẫn introspect?** Local verify nhanh và loại token sai; introspection lấy trạng thái account/role mới nhất. Đổi lại có thêm một network hop và DB read.
- **Nếu Auth chết?** Protected request hiện fail closed thành 401; availability bị phụ thuộc Auth. Có thể dùng cache ngắn, circuit breaker hợp lý, hoặc JWT thuần kèm cơ chế revoke/version tùy yêu cầu.
- **Tại sao HMAC nội bộ?** Không tin header identity do client tự gửi; Gateway ký để downstream xác minh nguồn và tính toàn vẹn. HMAC không thay thế network isolation/TLS/service identity.
- **Rate limit scale được không?** Chưa. `Map` chỉ đúng cho một Gateway. Nhiều replica cần Redis/ingress rate limiter và thuật toán atomic.

---

## 4. Các luồng nghiệp vụ và code

## 4.1. Đăng ký tài khoản và đồng bộ hồ sơ bằng Outbox

### Mục tiêu nghiệp vụ

Tạo credential ở Auth và cuối cùng tạo profile tương ứng ở User, không để mất sự kiện nếu RabbitMQ tạm thời lỗi.

### Luồng chi tiết

1. Client gọi `POST /api/auth/register` với email, password, username.
2. Gateway dùng `ValidationPipe` với `whitelist` và `transform`, sau đó proxy tới Auth.
3. Auth kiểm tra email tồn tại và hash password bằng bcrypt cost 10.
4. Trong **cùng MongoDB transaction**, Auth ghi credential và một outbox event chứa snapshot profile + `eventId` + `version`.
5. API trả thành công sau khi transaction commit; không cần chờ User service.
6. `OutboxPublisher` poll khoảng mỗi giây, claim tối đa 20 event bằng lease 30 giây.
7. Publisher gửi persistent/confirm message vào queue `user-profile-sync`. Nếu lỗi, event còn trong DB và retry exponential, tối đa khoảng 5 phút giữa hai lần thử.
8. User consumer xử lý theo `eventId` và `version` trong transaction. Version cũ/trùng bị bỏ qua; event DELETE để lại tombstone để event cũ không làm profile sống lại.
9. Kết quả là Auth mạnh nhất quán với credential; User đồng bộ theo **eventual consistency**.

```text
register
  -> transaction[credential + outbox]
  -> commit
  -> outbox publisher -> RabbitMQ
  -> User consumer -> transaction[sync_state + profile]
```

Code:

- [AuthService.register/createCredential/enqueueProfile](auth/src/modules/auth/auth.service.ts)
- [Outbox service](auth/src/modules/outbox/outbox.service.ts)
- [Outbox publisher](auth/src/modules/outbox/outbox.publisher.ts)
- [User consumer](user/src/modules/user/user-profile-sync.consumer.ts)
- [Profile sync/version](user/src/modules/user/profile-sync.service.ts)

### Câu trả lời phỏng vấn

> Vấn đề em muốn giải là dual write: nếu ghi Auth xong rồi publish RabbitMQ ngay mà broker lỗi thì User không bao giờ có profile. Em lưu credential và outbox event trong cùng transaction MongoDB. Worker publish sau và retry. Consumer phía User idempotent theo event ID/version, nên delivery ít nhất một lần không làm dữ liệu bị áp dụng hai lần. Đổi lại profile có thể trễ một khoảng ngắn, nên Auth là source of truth còn User là read model.

### Giới hạn cần thừa nhận

- MongoDB phải là replica set/sharded cluster để dùng transaction; Atlas đáp ứng điều này.
- Outbox cho **at-least-once**, không tự tạo exactly-once end-to-end.
- Sau register có một cửa sổ profile chưa xuất hiện; UI/API cần chịu được eventual consistency.
- Kiểm tra email trước insert vẫn có race; unique index mới là hàng rào cuối cùng và cần map duplicate-key thành lỗi nghiệp vụ rõ ràng.

## 4.2. Login bằng password → OTP email → JWT

### Luồng chi tiết

1. `POST /api/auth/login`: Auth tìm credential và `bcrypt.compare` password.
2. Email không tồn tại và sai password dùng cùng một thông báo để hạn chế account enumeration.
3. Redis key `otp:ratelimit:<email>` chặn gửi lại trong 60 giây.
4. OTP 6 số được tạo bằng `crypto.randomInt`, lưu Redis 5 phút; bộ đếm lần sai được reset.
5. Auth publish message `send-otp` qua RabbitMQ.
6. Mail consumer validate message rồi gửi SMTP. Lỗi tạm thời được chuyển qua retry queue có TTL; quá số lần thử hoặc message không hợp lệ vào DLQ.
7. Client gọi `POST /api/auth/verify`.
8. Auth so OTP và tăng counter nguyên tử có expiry. Sai tới 5 lần thì xóa OTP và khóa mã đó.
9. Đúng thì xóa OTP/counter, lấy username từ User theo kiểu best-effort và phát access + refresh token.

Code:

- [Auth login/verify](auth/src/modules/auth/auth.service.ts)
- [Redis operations](auth/src/modules/redis/redis.service.ts)
- [Rabbit publisher](auth/src/modules/rabbitmq/rabbitmq.service.ts)
- [Mail OTP consumer](mail/src/modules/mail/otp-mail.consumer.ts)
- [Mail Rabbit retry/DLQ](mail/src/modules/rabbitmq/rabbitmq.service.ts)

### Vì sao dùng RabbitMQ cho mail?

> Gửi SMTP là I/O bên ngoài, có thể chậm hoặc lỗi tạm thời. Queue tách việc tạo OTP khỏi worker gửi mail và cho phép retry/DLQ có kiểm soát. Trong code hiện tại, Auth vẫn đợi confirm publish để biết message đã vào broker; nó không đợi SMTP hoàn tất.

### Các tình huống lỗi

- RabbitMQ publish lỗi sau khi OTP đã lưu: login trả lỗi nhưng OTP/rate-limit có thể còn tới TTL. Đây là điểm có thể cải tiến bằng outbox hoặc cleanup khi publish fail.
- Mail lỗi: retry queue; quá giới hạn vào DLQ để điều tra, không requeue vô hạn nóng.
- Redis lỗi: luồng OTP/refresh fail closed thay vì bỏ qua kiểm soát.
- OTP hiện lưu plain text trong Redis. Bản tốt hơn là lưu HMAC/hash OTP và so sánh constant-time.

### “Đây có thật là 2FA không?”

> Nó là xác minh hai bước trong login: knowledge factor là password và mã dùng một lần gửi qua email. Em sẽ mô tả chính xác là email OTP second step; nó yếu hơn TOTP hoặc hardware factor vì email có thể cùng nằm trên thiết bị và OTP hiện chưa được hash. Nếu yêu cầu bảo mật cao em sẽ dùng TOTP/WebAuthn, hash OTP, thêm per-IP/per-account throttling và audit.

## 4.3. Access token, refresh token và rotation

1. Access token chứa user snapshot và `tokenType=access`; cấu hình hiện tại hết hạn 7 ngày.
2. Refresh token chứa `sub`, UUID `jti`, `tokenType=refresh`; hết hạn 30 ngày.
3. Redis chỉ giữ JTI refresh hiện hành theo user, nghĩa là thiết kế hiện cho một refresh session active/user.
4. Khi refresh, Auth verify token, kiểm tra credential còn tồn tại và dùng Lua/atomic compare-and-set để thay JTI cũ bằng JTI mới.
5. Hai request refresh đua nhau thì chỉ một request rotate thành công; token cũ bị coi là revoked/replaced.
6. Xóa account sẽ xóa refresh key.

Code: [issueSessionTokens/refreshToken](auth/src/modules/auth/auth.service.ts), [Redis rotate](auth/src/modules/redis/redis.service.ts).

Câu trả lời:

> Em không lưu cả refresh token mà lưu JTI hiện hành. Khi refresh, Redis chỉ rotate nếu JTI gửi lên đúng giá trị đang lưu; thao tác atomic ngăn replay/race. Điểm đánh đổi là đăng nhập ở thiết bị mới có thể thay phiên cũ. Nếu cần multi-device em sẽ lưu nhiều session có device ID, TTL, revoke riêng và reuse detection.

Điểm cải tiến nên chủ động nói: access token 7 ngày khá dài; hệ thống thật thường dùng access token ngắn hơn, ví dụ 10–30 phút, kết hợp refresh rotation.

## 4.4. Google sign-in

1. Client lấy Google ID token và gửi `POST /api/auth/login-google`.
2. Auth dùng `verifyIdToken` với `GOOGLE_WEB_CLIENT_ID` làm audience.
3. Chỉ chấp nhận email có `email_verified=true`.
4. Nếu credential chưa có, Auth tạo credential với random inaccessible password hash và cùng luồng Outbox tạo profile.
5. Nếu đã có, dùng account hiện hữu rồi phát session token của NRApp.

Code: [AuthService.loginWithGoogle](auth/src/modules/auth/auth.service.ts).

Trả lời chính xác:

> Backend của em không tin email do client tự gửi; nó verify chữ ký/claims của Google ID token và audience. Sau đó NRApp phát token riêng. Đây là Google sign-in dựa trên ID token. Nếu triển khai Authorization Code flow đầy đủ phía server thì còn code exchange, PKCE/client secret tùy loại client và quản lý Google refresh token — phần đó không có trong code hiện tại.

## 4.5. Thay đổi email/role/xóa account

- Auth là source of truth cho email, role và trạng thái tồn tại.
- Thay đổi được thực hiện trong transaction cùng một outbox event tăng `syncVersion`.
- Gateway introspection đọc credential hiện tại nên role cũ trong access token không được tin tuyệt đối.
- User áp event version mới; event cũ/trùng bị bỏ qua.
- Xóa account thu hồi refresh JTI và phát DELETE; User giữ sync tombstone để chống resurrection.

Code: [AuthService changeCredential](auth/src/modules/auth/auth.service.ts), [sync schema](user/src/schemas/profile-sync-state.schema.ts).

Lưu ý rất quan trọng: Auth hiện normalize mọi role khác `admin` về `user`. Nếu interviewer hỏi `manager/chef`, nói đó là role enum/route policy ở module nghiệp vụ cần được thống nhất lại với Auth trước khi dùng thật.

## 4.6. Mail retry và Dead Letter Queue

Luồng:

```text
send-otp queue
  -> gửi SMTP thành công -> ACK
  -> lỗi retryable -> publish confirm vào retry queue có TTL -> ACK message gốc
  -> TTL hết -> dead-letter quay lại queue chính
  -> quá max retry / message invalid -> DLQ -> ACK message gốc
```

Điểm hay để nói:

- Consumer chỉ ACK message gốc sau khi message thay thế đã publish confirm thành công.
- `prefetch` giới hạn số email đang xử lý; VPS dùng mức nhỏ để bảo vệ RAM/tài nguyên SMTP.
- DLQ tránh poison message lặp vô hạn.
- At-least-once có thể gửi trùng nếu crash đúng thời điểm; email OTP nên có idempotency/deduplication nếu yêu cầu nghiêm ngặt.

## 4.7. Chat REST và Socket.IO

### Tạo conversation

1. Client gửi ID người còn lại.
2. Chat validate ObjectId, không cho chat với chính mình và hỏi User để chắc user tồn tại.
3. Query conversation có đúng hai participant bằng `$all` + `$size`.
4. Có thì trả conversation cũ; chưa có thì tạo mới.

Điểm race: schema hiện chưa có canonical pair unique key, nên hai request đồng thời vẫn có thể tạo conversation trùng. Cách sửa là lưu `participantKey` đã sort rồi unique index, hoặc transaction/locking phù hợp.

### Gửi message

1. Gateway và Chat kiểm MIME/kích thước ảnh, giới hạn khoảng 5 MB.
2. Chat xác minh sender thuộc conversation và message có text hoặc image.
3. Ảnh được upload Cloudinary, transform giới hạn kích thước.
4. Lưu Message, cập nhật `latestMessage/updatedAt` của Chat.
5. Nếu cập nhật Chat lỗi, service xóa message/ảnh theo kiểu best-effort compensation.
6. Socket Gateway phát `newMessage` đến toàn bộ socket của recipient đang online.

### Nhận message/seen/presence

- Socket handshake gửi token trong `handshake.auth.token`; server verify JWT.
- Map `userId -> Set<socketId>` hỗ trợ một user nhiều thiết bị/tab.
- `typing`, `typingStop`, `newMessage`, `messagesSeen` là event realtime.
- REST lấy message kiểm membership, đánh dấu message phía kia đã xem rồi emit `messagesSeen`.

Code:

- [Chat service](chat/src/modules/chat/chat.service.ts)
- [Socket gateway](chat/src/modules/chat/chat.gateway.ts)
- [Image service](chat/src/modules/chat/chat-image.service.ts)
- [Chat schema](chat/src/schemas/chat.schema.ts), [Message schema](chat/src/schemas/message.schema.ts)

### Câu trả lời phỏng vấn

> REST xử lý thao tác cần response rõ như tạo chat, gửi/lấy lịch sử; Socket.IO dùng cho server push và trạng thái realtime. Gateway proxy cả polling và WebSocket upgrade tới Chat. Một user có thể có nhiều socket nên em giữ Set thay vì một socket ID. Thiết kế hiện là single Chat instance; scale ngang cần Redis adapter/message broker và sticky session hoặc transport config phù hợp.

### Giới hạn đáng nói

- Presence đang ở memory, restart làm mất trạng thái tạm thời.
- Socket verify JWT cục bộ nhưng chưa introspect account hiện tại như REST.
- Event typing cần kiểm membership/authorization chặt hơn.
- Lấy toàn bộ message chưa có cursor pagination.
- Danh sách chat gọi User để enrich từng conversation, có nguy cơ N+1; nên batch.

## 4.8. Todo — phân quyền và state machine

Nghiệp vụ chính:

- Management tạo, sửa, giao việc, xóa và xem danh sách.
- User xem `my-tasks`, xem chi tiết việc mình được giao và cập nhật status trong phạm vi cho phép.
- Khi giao việc, Todo hỏi User service để đảm bảo assignee tồn tại.
- Danh sách có pagination/filter/search và enrich user theo batch thay vì gọi từng ID.

State transition:

| Người thao tác | Chuyển trạng thái hợp lệ                                                                                          |
| -------------- | ----------------------------------------------------------------------------------------------------------------- |
| Assignee       | `todo -> in_progress -> done`                                                                                     |
| Management     | `todo -> in_progress/cancelled`; `in_progress -> todo/done/cancelled`; `done -> in_progress`; `cancelled -> todo` |

Code: [TaskService](todo/src/modules/task/task.service.ts), [TaskController](todo/src/modules/task/task.controller.ts), [Task schema](todo/src/schemas/task.schema.ts).

Race condition:

> Khi đổi status hoặc assign, service dùng conditional query chứa trạng thái/assignee mong đợi thay vì đọc rồi save vô điều kiện. Nếu request khác đã đổi dữ liệu, update không match và request sau phải nhận conflict. Đây là optimistic concurrency ở mức nghiệp vụ, tránh lost update.

Fallback:

> Nếu User lookup để enrich directory lỗi, danh sách task vẫn có thể trả ID thô; nhưng lúc tạo/giao việc thì validate user là yêu cầu nghiệp vụ nên phải fail. Em phân biệt dependency bắt buộc với enrichment best-effort.

## 4.9. WorkSchedule — đăng ký lịch tháng và phê duyệt

### Tạo đăng ký lịch

1. User gửi `month=YYYY-MM` và các entry ngày làm.
2. Validator kiểm ngày thuộc tháng, không trùng, loại `office/remote/day_off/leave`, ca `full_day/morning/afternoon`, note giới hạn độ dài.
3. Policy singleton quyết định cửa sổ đăng ký và trạng thái lock.
4. Service không cho sửa ngày quá khứ hoặc đè dữ liệu legacy không hợp lệ.
5. Trong transaction, tạo `ScheduleEntry` và `ScheduleRequest` pending.
6. Unique partial index bảo đảm một employee/tháng cho dữ liệu mới.

### Duyệt/từ chối/resubmit

- Management duyệt, từ chối hoặc bulk approve.
- Rejected request chỉ owner mới resubmit; resubmit reset review fields và dùng điều kiện status để tránh race.
- Khi approved, lịch `remote` tạo attendance tự động; cập nhật lịch approved sẽ đồng bộ/tháo attendance liên quan trong cùng transaction.
- Với ngày quá khứ, không cho thêm/sửa/xóa lịch để giữ tính lịch sử.

Code:

- [Schedule service](workschedule/src/modules/schedule/schedule.service.ts)
- [Entry validator](workschedule/src/modules/schedule/utils/schedule-entry-validator.ts)
- [Policy service](workschedule/src/modules/policy/policy.service.ts)
- [Schedule request schema](workschedule/src/schemas/schedule-request.schema.ts)

Câu trả lời:

> Luồng lịch có nhiều document liên quan nên em dùng MongoDB transaction với snapshot/majority. Index unique employee + month là invariant ở DB, còn validator xử lý lỗi nghiệp vụ dễ hiểu. Em không chỉ kiểm tra ở application vì hai request đồng thời vẫn có thể vượt qua bước đọc.

## 4.10. Work request — nghỉ, đi muộn, OT, công tác, remote

1. DTO nhận loại request, thời gian, ca, lý do, attachment và field riêng theo loại.
2. Validator yêu cầu end time cho overtime/business trip, location cho business trip, project cho overtime; tối đa 5 attachment.
3. Không cho duplicate request cùng loại/thời điểm khi đang pending/approved.
4. User chỉ hủy request pending của mình.
5. Management duyệt/từ chối bằng conditional status; reject phải có lý do.

Code: [WorkRequestService](workschedule/src/modules/work-request/work-request.service.ts), [validator](workschedule/src/modules/work-request/utils/work-request-validator.ts).

Nếu hỏi “duplicate check có tuyệt đối không?”:

> Check ở application cải thiện UX nhưng nếu không có unique/partial index tương ứng thì vẫn có race. Với invariant cứng em sẽ thiết kế một dedupe key và unique partial index cho các trạng thái active, rồi bắt duplicate-key.

## 4.11. Chấm công bằng QR

1. Management sinh token ngẫu nhiên 32 byte; token hết hạn khoảng 30 giây và collection có TTL index.
2. User scan QR; service lấy ngày theo múi giờ Việt Nam.
3. Chỉ user có lịch `office` đã approved cho ngày đó mới được chấm công.
4. Scan đầu tạo check-in; scan token khác sau đó tạo check-out.
5. Unique index theo employee/date/source và conditional upsert/update xử lý hai request scan đồng thời.
6. Lịch remote đã approved được đồng bộ attendance tự động, không dùng QR office.

Code: [AttendanceService](workschedule/src/modules/attendance/attendance.service.ts), [QR token schema](workschedule/src/schemas/attendance-qr-token.schema.ts), [attendance schema](workschedule/src/schemas/attendance-record.schema.ts), [Vietnam time helpers](workschedule/src/utils/vietnam-time.ts).

Điểm bảo mật:

- Token phải khó đoán, TTL ngắn, một lần sử dụng theo ngữ cảnh.
- QR động giảm việc chụp gửi lại nhưng chưa chứng minh người thật ở đúng vị trí; có thể bổ sung device attestation/geofence tùy chính sách riêng tư.
- Không dùng giờ client làm nguồn sự thật.

## 4.12. Canteen — menu, undo/redo, bàn và tạo đơn

### Menu và undo/redo

- Public chỉ thấy category active và món available; admin thấy cả dữ liệu ẩn.
- Tạo/sửa/xóa menu lưu command cũ/mới vào Redis theo từng admin.
- Mỗi stack giới hạn khoảng 50 command, TTL 24 giờ; thao tác mới xóa redo stack.
- Undo/redo thực hiện inverse operation trên MongoDB.

Code: [MenuService](canteen/src/modules/menu/menu.service.ts), [MenuHistoryManager](canteen/src/modules/menu/utils/undo-stack.ts).

Trade-off:

> Undo/redo này là tiện ích vận hành ngắn hạn, không phải audit log bất biến. Redis mất dữ liệu thì lịch sử undo mất nhưng menu chính trong MongoDB vẫn còn. Nhiều admin sửa cùng item vẫn cần optimistic version/audit nếu yêu cầu chặt.

### Tạo đơn

1. User chọn bàn, món, số lượng và option; phương thức active chỉ là `CASH`.
2. Server batch-load menu/category từ DB, không tin tên/giá client gửi.
3. Server từ chối món unavailable/category inactive, map option theo tên chuẩn hóa và tự tính giá bằng integer VND.
4. `findOneAndUpdate` atomically đổi bàn `empty|occupied -> occupied`; bàn `reserved` bị từ chối.
5. Counter tạo order number dạng `#1001...`, xử lý duplicate/race.
6. Order lưu snapshot tên món, option và đơn giá để lịch sử không đổi khi menu về sau thay đổi.
7. Nếu save order lỗi, service compensation trả bàn về empty nếu không còn order chưa thanh toán.

Code: [OrderService](canteen/src/modules/order/order.service.ts), [Order schema](canteen/src/schemas/orders.schema.ts), [TableService](canteen/src/modules/table/table.service.ts).

### Hủy và thanh toán tiền mặt

- Owner hoặc admin được hủy; chỉ đơn `CREATED`/payment `PENDING`, hủy lại trả kết quả idempotent.
- Admin xác nhận cash bằng conditional update `PENDING -> PAID` và `CREATED -> COMPLETED`.
- Conditional filter ngăn pay/cancel cùng lúc cùng thắng.
- Sau hủy/pay, service reconcile bàn về empty chỉ khi không còn order active chưa thanh toán.

Câu trả lời:

> Em luôn tính giá ở server từ menu hiện hành và lưu snapshot vào order; client không thể tự gửi `finalAmount`. Với pay/cancel em dùng conditional atomic update, không đọc status rồi save, nên hai request đua nhau chỉ một trạng thái hợp lệ thắng. Phần giữ bàn và lưu order hiện dùng compensation chứ chưa phải transaction đa document; đó là một trade-off em sẽ nâng cấp nếu yêu cầu consistency cao hơn.

## 4.13. Payment/VietQR — code có nhưng chưa chạy active

Luồng được thiết kế trong source:

1. Gateway lấy `finalAmount` đáng tin từ Canteen rồi gọi Payment bằng HMAC identity.
2. Payment tạo hoặc reuse intent `PENDING` trong PostgreSQL và trả VietQR.
3. Casso gửi webhook; service verify HMAC-SHA512.
4. Dùng unique provider event và row lock để idempotent, kiểm account/amount.
5. Cập nhật `SUCCESS` và ghi SQL outbox trong cùng transaction.
6. Outbox publish `payment.succeeded.v1`; Canteen sẽ cập nhật order paid.

Code: [Payment README](payment/README.md), [PaymentService](payment/src/modules/payment/payment.service.ts), [Casso signature](payment/src/modules/payment/casso-signature.service.ts), [Payment outbox](payment/src/modules/outbox/outbox.publisher.ts).

Cách nói bắt buộc:

> Em đã implement/stage Payment service dùng PostgreSQL, Casso webhook và outbox, nhưng tại thời điểm CV/VPS hiện tại Casso chưa khả dụng nên Compose đặt nó ở profile `payment-later`; Gateway active cũng chưa import Payment module và Canteen active chỉ nhận cash. Vì vậy em trình bày đây là code đã xây, chưa nhận là luồng production đang chạy.

---

## 5. Hạ tầng và vận hành

## 5.1. Nginx → Gateway → service

- Nginx kết thúc HTTPS và forward vào một Gateway ở loopback/container network.
- Gateway `trust proxy = 1` để lấy đúng IP client qua đúng một proxy.
- REST dùng Nest HTTP client với timeout, không follow redirect tùy tiện và map lỗi transport thành 502.
- `/socket.io` được proxy riêng, gồm cả HTTP polling và WebSocket upgrade.
- Các port nội bộ bind loopback hoặc chỉ ở Docker network; không nên expose toàn bộ service ra Internet.

Nếu hỏi Cloudflare:

> Cloudflare/Nginx thuộc lớp edge/DNS/TLS của deployment. Em chỉ mô tả đúng cấu hình mình đã thực sự bật; không gọi Cloudflare là load balancer hay WAF nếu chưa cấu hình và kiểm chứng các chức năng đó.

## 5.2. CI/CD và rollback

### CI

Mỗi service gọi reusable workflow dùng Node 22:

1. Checkout đúng source và shared observability package ở immutable SHA.
2. `npm ci`.
3. Chặn critical production dependency vulnerability.
4. Lint, format check, unit test, build.

### CD

1. Chỉ chấp nhận tám service allowlist, full 40-character SHA và revision vẫn là head của default branch.
2. Build image `linux/amd64` đúng revision đã qua CI.
3. Nén, kiểm gzip/kích thước rồi truyền bằng SSH key bị giới hạn quyền và pinned host key.
4. Receiver trên VPS validate command, service, SHA, archive/tag/architecture.
5. `flock` serialize đoạn thay container; image cũ được tag `rollback`.
6. Compose force-recreate đúng một service và `--wait --wait-timeout 300`.
7. Verify container đang chạy đúng image. Nếu không healthy, tag lại image cũ và redeploy rollback.

Code:

- [Reusable CI](logger/.github/workflows/reusable-node-ci.yml)
- [Reusable CD](logger/.github/workflows/reusable-vps-cd.yml)
- [VPS receiver](logger/deploy/vps-ci-receiver)

Câu trả lời:

> Rollback của em là image-level rollback trên một VPS: trước deploy tag image hiện hành, sau đó recreate và đợi healthcheck. Nếu rollout lỗi thì khôi phục image cũ. Nó chưa giải quyết rollback database migration; migration phải backward-compatible hoặc có chiến lược expand/contract riêng.

## 5.3. Health check, log và request ID

- Liveness trả lời process sống; readiness ở service có DB/broker kiểm dependency cần thiết.
- Docker Compose dùng healthcheck để CD quyết định rollout thành công.
- Shared package dùng Pino log JSON, Nest adapter, redaction field nhạy cảm và AsyncLocalStorage để giữ context.
- Request ID từ bên ngoài được coi là untrusted; middleware tạo/canonicalize correlation ID.
- Gateway ghi route, status, duration và classification; lỗi bất ngờ có `errorId`, nhưng không log password/token/body nhạy cảm.
- Docker local logging driver rotate file theo size/count, tránh đầy disk.

Code: [observability README](logger/packages/observability/README.md), [request outcome](gateway/src/common/middleware/request-outcome.middleware.ts), [VPS Compose limits](compose.vps.yaml).

## 5.4. Monitoring

Stack active:

- node_exporter đọc CPU, RAM, filesystem của host.
- Prometheus scrape `node-exporter` và chính Prometheus mỗi 60 giây.
- Grafana dùng Prometheus datasource và dashboard provision sẵn.
- Prometheus giữ tối đa khoảng 2 ngày/512 MB trong cấu hình VPS tối giản.
- Container có healthcheck và memory limit; app log JSON để điều tra bằng Docker logs.

Code: [monitoring Compose](logger/compose.yaml), [Prometheus config](logger/vps-minimal/prometheus.yaml).

Không nói quá:

> Metrics active hiện chủ yếu là host/resource metrics. OTel application metrics, distributed tracing, Loki và Alertmanager không nằm trong stack đang chạy được xác minh. Request ID cho phép correlation log thủ công nhưng không tương đương distributed trace.

## 5.5. Backup và restore

### Luồng thật

1. Systemd user timer chạy khoảng 02:30 Việt Nam, random delay tối đa 5 phút, `Persistent=true`.
2. Python worker lấy Mongo URI/DB từ container User mà không log credential.
3. Dùng container Mongo tool tạm để `mongodump` live từng collection.
4. Dùng `redis-cli --rdb` và `redis-check-rdb` để tạo/kiểm Redis snapshot.
5. Export RabbitMQ definitions; nếu Payment PostgreSQL đang chạy mới thêm `pg_dump`.
6. Đóng gói manifest, config và `SHA256SUMS`; mã hóa toàn archive bằng Age trước khi rời VPS.
7. Chỉ sau backup thành công mới prune: giữ ít nhất 7 bản và chỉ xóa bản quá 14 ngày.
8. GitHub Actions khoảng 03:15 lấy archive qua forced SSH command, kiểm SHA-256, upload artifact 30 ngày và commit bản mã hóa.

Code/tài liệu:

- [Backup README](../nrapp-backup/README.md)
- [backup.py](../nrapp-backup/scripts/backup.py)
- [systemd timer](../nrapp-backup/systemd/nrapp-backup.timer)
- [off-site workflow](../nrapp-backup/.github/workflows/backup.yml)
- [kết quả kiểm chứng](../docBEMD/NOI_DUNG_CV_HA_TANG_NRAPP_VPS.md)

### Những gì đã kiểm chứng và chưa kiểm chứng

- Đã tải archive off-site, verify checksum, giải mã và restore MongoDB vào container tách biệt: 19 collections, 4 documents, 53 indexes, 0 document failed tại lần diễn tập.
- Redis RDB đã qua `redis-check-rdb` nhưng chưa diễn tập khôi phục toàn hệ thống.
- RabbitMQ backup chỉ là definitions, **không chứa message đang chờ**.
- Không backup media Cloudinary, metrics Prometheus/Grafana hay snapshot toàn bộ VPS.
- MongoDB Atlas Free dump là live per collection, không phải point-in-time snapshot toàn hệ thống; dữ liệu ít trong lần test không chứng minh RTO cho database lớn.

Câu trả lời:

> Em phân biệt backup với restore-tested backup. Job tạo dump, checksum và Age-encrypt; khóa giải mã không nằm trên VPS/GitHub. Em đã diễn tập Mongo restore vào môi trường tách biệt, không restore thẳng production. RPO mục tiêu theo lịch khoảng 24 giờ; em chưa công bố RTO production vì dữ liệu test còn nhỏ và chưa phục hồi full stack.

## 5.6. Load test k6 — phần dễ bị hỏi vặn nhất

Ba API đọc trong script:

1. `GET /api/user/me`
2. `GET /api/todo/my-tasks`
3. `GET /api/chat/chat/all`

Điều kiện cần nhớ:

- Backend một VPS 2 vCPU/~4 GB; MongoDB Atlas Free ở ngoài VPS.
- 50 virtual users dùng **chung một access token/tài khoản**.
- Đây là REST read-only; không đại diện write, WebSocket, nhiều user hay toàn hệ thống.
- `VU` là concurrency, không phải RPS.
- 100% success chỉ nói request/check thành công, không nói latency đạt mục tiêu.

Bằng chứng hiện có trong [LOADTEST.md](scripts/LOADTEST.md): probe sau sửa Auth giữ 50 VU trong 60 giây ghi nhận 26.45 RPS, 100% success, p95 khoảng 1399.94 ms và p99 khoảng 1698.68 ms; **latency threshold bị trượt**. Tài liệu có recipe giữ 3 phút nhưng không có raw result 3 phút 26.5 RPS trong workspace.

Câu trả lời an toàn nếu CV đã gửi:

> Bài test của em chạy ba API đọc tuần tự, 50 VU dùng chung một account, nên con số khoảng 26.5 RPS chỉ có ý nghĩa trong đúng workload đó. Request success là 100%, nhưng tail latency chưa đạt mục tiêu 500 ms; lần probe được lưu trong repo có p95 khoảng 1.4 giây, p99 khoảng 1.7 giây. Em không dùng kết quả này để tuyên bố capacity production. CV ghi sustained 3 phút; trước khi khẳng định mốc đó em cần đưa raw report tương ứng, còn bằng chứng source hiện tại em chắc chắn là lượt giữ 60 giây.

Vì sao Auth là điểm đáng chú ý:

- Mỗi protected request có thêm introspection.
- Auth lại đọc credential MongoDB Atlas; nhiều VU chung user gây các read giống nhau.
- `InFlightReads` coalesce request đồng thời và optional TTL rất ngắn giảm duplicate reads trên một instance.
- Kết quả vẫn chịu network đến Atlas, tài nguyên VPS và máy phát tải; không được kết luận DB là bottleneck duy nhất khi chưa có profiling đầy đủ.

Nếu hỏi cách test tốt hơn:

- Ghi commit/image/config/rate limit và dataset của từng run.
- Dùng token/tài khoản khác nhau, dataset có task/chat thật.
- Tách smoke, load, stress, soak; warm-up rồi giữ đủ dài.
- Đo throughput, error rate, p50/p95/p99 theo endpoint và host/container/DB metrics.
- Test write và Socket.IO riêng, có cleanup/idempotency.
- Chạy generator ngoài VPS để không tranh CPU, đồng thời đo riêng đường nội bộ để phân tích network.

---

## 6. Bảng đối chiếu từng ý trong CV

| Ý trong CV                        | Bằng chứng/code                     | Cách trả lời                                                                           | Ranh giới                                            |
| --------------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| 8 Node 22/NestJS 11 microservices | `compose.vps.yaml`, trạng thái VPS  | Kể đúng tám tên service                                                                | Payment thứ chín chưa active                         |
| MongoDB, Redis, RabbitMQ          | Compose và service modules          | MongoDB cho domain data; Redis cho ephemeral state; Rabbit cho async mail/profile sync | Mongo ở Atlas ngoài Compose                          |
| Nginx HTTPS → Gateway             | Gateway bootstrap/deploy docs       | Một public entry, proxy REST + Socket                                                  | Một Gateway, chưa HA/load balance                    |
| JWT/RBAC/validation               | JWT strategy, guards, DTO pipe      | Local verify + Auth introspection; decorator/guard                                     | Auth active chỉ admin/user                           |
| Rate limiting                     | Gateway middleware + OTP Redis keys | Gateway per-IP memory; OTP per-email Redis                                             | Không nói Gateway limiter dùng Redis                 |
| OAuth2                            | Google ID token verify              | Google sign-in, verified token/audience                                                | Không tự nhận full auth-code flow                    |
| 2FA                               | Password + email OTP                | Second-step email OTP                                                                  | Không phải TOTP/WebAuthn                             |
| Socket.IO realtime                | Chat gateway                        | multi-socket user, message/seen/typing                                                 | Single instance, chưa Redis adapter                  |
| Redis cache                       | Auth InFlight + Redis usages        | OTP/JTI/rate key/undo stack; Auth identity micro-cache in memory                       | Không phải general-purpose distributed cache         |
| Prometheus/Grafana                | logger minimal stack                | Host CPU/RAM/disk scrape 60s                                                           | Chưa app tracing/alert stack active                  |
| CI/CD rollback                    | reusable workflows + receiver       | exact SHA image, wait health, rollback image                                           | Không rollback DB migration                          |
| JSON log/request ID               | shared observability                | correlation across hop, redaction                                                      | Không bằng distributed tracing                       |
| Backup                            | `nrapp-backup` repo                 | timer, dump, checksum, Age, off-site                                                   | live Mongo dump; Rabbit definitions only             |
| k6 ~26.5 RPS                      | `scripts/LOADTEST.md`               | exact workload and success                                                             | latency fail; 3-minute result chưa thấy raw evidence |
| PostgreSQL                        | Payment source/Compose              | đã implement trong Payment staged                                                      | chưa chạy trên VPS active                            |
| MySQL                             | Không thấy trong NRApp              | chỉ nói mức học/bài khác nếu có                                                        | không gắn vào NRApp                                  |
| Cloudinary                        | Chat image service                  | upload ảnh chat + validation/resize                                                    | media không có trong backup hiện tại                 |

---

## 7. Bộ câu hỏi HR và câu trả lời mẫu

Các câu dưới đây là khung nói, không học thuộc từng chữ. Thay chi tiết bằng trải nghiệm thật nếu khác.

### 7.1. “Hãy giới thiệu về bản thân”

Dùng bài 60 giây ở đầu tài liệu. Kết thúc bằng vị trí muốn học và giá trị có thể đóng góp, không kể lại toàn bộ CV.

### 7.2. “Tại sao em chọn Backend?”

> Em thích phần biến yêu cầu nghiệp vụ thành invariant rõ ràng và xử lý các trường hợp không lý tưởng: request đồng thời, dependency lỗi, retry, quyền truy cập và quan sát production. Khi làm NRApp, phần khiến em hứng thú nhất không phải chỉ tạo CRUD mà là làm sao register không mất profile event, refresh token không replay và pay/cancel không cùng thắng. Vì vậy em muốn đi sâu Backend.

### 7.3. “Dự án này em làm một mình hay theo nhóm?”

> Đây là personal project nên em là người chịu trách nhiệm chính từ phân tích, code đến deploy. Ưu điểm là em hiểu end-to-end; hạn chế là chưa có đủ va chạm với code review và phối hợp team lớn. Vì vậy em đang tìm môi trường có pull request, review, ticket và feedback thật để cải thiện cách làm việc nhóm.

Không bịa số thành viên, vai trò leader hay người dùng production.

### 7.4. “Điểm mạnh của em là gì?”

> Điểm mạnh của em là theo vấn đề đến lúc chạy được và kiểm chứng được. Ví dụ em không dừng ở việc viết Dockerfile: em thêm healthcheck, deploy đúng commit, lưu image cũ để rollback, rồi kiểm tra monitoring và backup/restore. Em cũng cố gắng ghi rõ giới hạn thay vì lấy một con số test rồi suy rộng cho toàn hệ thống.

### 7.5. “Điểm yếu của em là gì?”

> Vì dự án cá nhân, có lúc em mở rộng quá nhiều service trước khi chốt metric và test cho luồng cốt lõi. Em đã nhận ra điều đó khi load test: success 100% nhưng p99 vẫn cao. Hiện em khắc phục bằng cách đặt acceptance criteria trước, đo từng hop, ưu tiên bottleneck và ghi lại giới hạn của kết quả.

Điểm yếu này thật, có hành động sửa và không tự loại mình khỏi vị trí.

### 7.6. “Thất bại/lỗi khó nhất?” — STAR số 1

**Situation:** Protected API qua Gateway có tail latency cao khi tăng VU.

**Task:** Xác định chậm ở Internet/Nginx/Gateway/Auth/User/DB chứ không đoán.

**Action:** Chạy k6 cùng workload, probe từng chặng, thêm timing Auth, so sánh từ client/VPS/Nginx loopback; nhận ra mỗi request introspect và nhiều request cùng đọc một credential. Thêm coalescing/cache identity rất ngắn cho một Auth instance, vẫn verify JWT mỗi request và invalidation khi credential đổi.

**Result:** Probe 50 VU tăng từ khoảng 24.12 lên 26.45 RPS, success 100%, nhưng p95/p99 vẫn trượt. Kết luận đúng không phải “đã tối ưu xong”, mà là giảm duplicate read và xác định còn tail latency/network/Atlas cần đo tiếp.

### 7.7. “Một quyết định kỹ thuật em tự hào?” — STAR số 2

> Khi tách Auth và User, em thấy ghi credential rồi publish trực tiếp có thể mất profile event. Em chuyển sang transactional outbox: credential và event commit cùng nhau, publisher retry, consumer idempotent theo event/version và giữ tombstone cho delete. Kết quả là broker downtime không làm mất sự kiện; trade-off là User eventual consistent. Đây là phần giúp em hiểu consistency thực tế hơn CRUD thông thường.

### 7.8. “Có mâu thuẫn/khác ý kiến trong team không?”

Vì là dự án cá nhân, không bịa conflict team:

> Dự án chính là cá nhân nên em chưa có tình huống mâu thuẫn lớn trong team để kể trung thực. Trong bài tập/hoạt động nhóm, em thường xử lý khác biệt bằng cách chốt yêu cầu, liệt kê phương án và tiêu chí đo rồi thử nhỏ. Em muốn được rèn thêm kỹ năng review và trao đổi thiết kế trong môi trường công ty.

Nếu có trải nghiệm nhóm thật, thay bằng STAR thật của bạn.

### 7.9. “Tại sao chọn công ty chúng tôi?”

Trước phỏng vấn thay ba chỗ trong ngoặc:

> Em chọn công ty vì [sản phẩm/lĩnh vực cụ thể], vị trí có dùng [stack hoặc loại bài toán], và em thấy mô tả có [review/mentor/quy trình] phù hợp giai đoạn của em. Em có nền tảng NestJS, database, messaging và deploy từ NRApp; em muốn dùng nền tảng đó để đóng góp sớm, đồng thời học cách một team vận hành hệ thống thật có người dùng.

Không nói chung chung “công ty lớn, môi trường năng động”.

### 7.10. “Vì sao chúng tôi nên chọn em?”

> Em chưa có nhiều năm kinh nghiệm, nhưng em đã tự đưa một hệ thống backend qua đủ vòng: nghiệp vụ, bảo mật, async messaging, deploy, quan sát, backup và test tải. Em có thể đọc code, debug có số đo và thừa nhận phần chưa biết. Với vị trí intern/fresher, em tin mình có nền đủ để nhận task sớm và tốc độ học tốt khi có review.

### 7.11. “Mục tiêu 1–2 năm?”

> Trong 6 tháng đầu em muốn làm chắc coding convention, test, SQL/NoSQL, review và quy trình release của team. Sau 1–2 năm, mục tiêu là tự phụ trách được một module backend từ yêu cầu đến vận hành, biết thiết kế theo traffic/SLA thật chứ không chọn công nghệ theo tên.

### 7.12. “Em còn đi học, sắp xếp thời gian thế nào?”

> Em sẽ cung cấp lịch học cụ thể và cam kết số buổi/giờ đúng với JD. Em ưu tiên báo trước lịch thi, dùng calendar và chia task nhỏ có deadline. Em không muốn hứa full-time nếu lịch học không cho phép; em muốn chốt kỳ vọng rõ từ đầu để không ảnh hưởng team.

### 7.13. “Mức lương mong muốn?”

> Với vị trí intern/fresher, ưu tiên của em là phạm vi công việc, mentor và cơ hội làm backend thật. Em mong mức trong khung công ty dành cho vị trí này; nếu cần con số, em xin trao đổi theo tổng thời gian làm việc và trách nhiệm cụ thể. Em linh hoạt nếu hai bên thống nhất rõ kỳ vọng và review sau thời gian thử việc.

Nếu HR bắt buộc con số, chuẩn bị một khoảng dựa trên JD/thị trường trước buổi phỏng vấn; không bịa trong tài liệu này.

### 7.14. “Khi không biết một câu kỹ thuật, em làm gì?”

> Em sẽ nói phần em biết và giả định đang dùng, không đoán thành sự thật. Sau đó em tách vấn đề, đọc official docs/source, tạo reproduction nhỏ, thêm log/metric và kiểm chứng. Ví dụ với load test em không kết luận bottleneck chỉ từ CPU mà đo từng chặng và vẫn ghi rõ latency chưa đạt.

---

## 8. Câu hỏi kỹ thuật bám sát CV

### 8.1. REST khác WebSocket thế nào?

REST là request/response, stateless ở tầng HTTP, dễ cache/quan sát và hợp CRUD. WebSocket giữ kết nối hai chiều để server push với overhead thấp hơn polling, hợp chat/presence. Trong NRApp, lịch sử và mutation chính đi REST; notification realtime đi Socket.IO. Socket.IO còn có handshake, event protocol, reconnect và fallback transport, không đồng nghĩa raw WebSocket.

### 8.2. JWT có ưu/nhược gì?

JWT tự chứa claim, verify nhanh và thuận tiện qua service. Nhược điểm là khó revoke ngay, claim có thể stale, token bị lộ dùng được tới expiry và payload chỉ encode chứ không encrypt. NRApp khắc phục một phần bằng Auth introspection và refresh JTI, đổi lại tăng dependency/latency.

### 8.3. Authentication và authorization?

Authentication xác định bạn là ai; authorization xác định bạn được làm gì. JWT/OTP/Google thuộc authentication. Role guard, ownership check và state transition thuộc authorization. Chỉ kiểm role ở Gateway chưa đủ cho rule theo dữ liệu; service vẫn phải kiểm owner/member/assignee.

### 8.4. RBAC hoạt động thế nào?

Route khai báo role metadata, guard so role đã xác thực. Nhưng RBAC chỉ là tầng thô: Chat còn kiểm participant, Todo kiểm assignee, Canteen kiểm owner. Hiện enum role giữa Auth và domain chưa thống nhất; hợp đồng active chỉ admin/user là một technical debt cần sửa.

### 8.5. CORS có phải cơ chế bảo vệ API khỏi mọi client không?

Không. CORS là policy trình duyệt; curl/mobile/backend khác không bị chặn như browser. API vẫn cần authentication, authorization, validation, rate limit và network controls. Gateway hiện `origin: *`, `credentials: false`; nếu web production dùng cookie/credential thì phải allowlist origin cụ thể.

### 8.6. ValidationPipe `whitelist` làm gì?

Loại property không khai báo trong DTO và transform input sang type mong đợi. Nó giảm mass assignment/input rác nhưng không thay authorization hay business validation. Có thể bật `forbidNonWhitelisted` nếu muốn reject thay vì strip.

### 8.7. Redis dùng khi nào, MongoDB dùng khi nào?

Redis hợp dữ liệu ngắn hạn/atomic nhanh có TTL: OTP, attempt, refresh JTI, rate key, undo stack. MongoDB giữ business state cần bền vững. Không dùng Redis làm nguồn sự thật của order/profile trong thiết kế hiện tại.

### 8.8. RabbitMQ khác HTTP sync?

HTTP cho response tức thời và coupling theo availability. RabbitMQ buffer công việc, retry và tách producer/consumer nhưng đem lại duplicate, ordering và observability challenges. NRApp dùng HTTP khi cần kết quả ngay (validate user), Rabbit cho email/profile sync.

### 8.9. At-least-once và idempotency?

At-least-once nghĩa là message không dễ mất nhưng có thể nhận nhiều lần. Consumer phải khiến xử lý lại có cùng kết quả: lưu event ID/version, dùng unique key hoặc conditional update. ACK đúng thời điểm quan trọng; ACK trước khi commit có thể mất dữ liệu, ACK sau commit có thể redeliver nên vẫn cần idempotency.

### 8.10. Outbox pattern giải quyết gì, không giải quyết gì?

Giải quyết atomicity giữa ghi DB và ý định publish bằng cách lưu event cùng transaction. Nó không tự bảo đảm consumer xử lý đúng một lần, không bảo đảm ordering toàn cục và cần cleanup/monitor backlog. Consumer vẫn phải idempotent.

### 8.11. MongoDB transaction khi nào cần?

Một document update vốn atomic. Transaction dùng khi invariant trải nhiều document/collection, như credential + outbox hay schedule + request + attendance. Transaction có cost và cần replica set; không nên bọc mọi query theo thói quen.

### 8.12. Index để làm gì? Nhược điểm?

Index giảm số document scan và có thể thực thi uniqueness. Đổi lại tốn disk/RAM và làm write chậm hơn. Chọn index theo query filter/sort thật. Ví dụ Message có `(chat, createdAt)`, Task có `(assignedTo, createdAt)`, schedule có unique `(employee, month)` cho dữ liệu mới.

### 8.13. MongoDB và PostgreSQL khác nhau thế nào trong dự án?

MongoDB hợp document domain và iteration nhanh của NRApp; transaction/index vẫn được dùng cho invariant. Payment chọn PostgreSQL vì trạng thái tài chính, unique provider event, row lock và transaction relational rõ. Nhưng Payment chưa active. Không nói NoSQL “không có schema” — Mongoose vẫn có schema/validation.

### 8.14. Vì sao tiền dùng integer?

Tránh sai số floating point. Giá VND được tính bằng safe integer ở server. Với hệ tiền có phần thập phân, lưu minor unit như cents hoặc decimal/numeric trong DB tùy yêu cầu.

### 8.15. Optimistic concurrency là gì?

Update kèm điều kiện phiên bản/trạng thái cũ. Nếu dữ liệu đã bị request khác đổi, `matchedCount=0`, trả conflict/thử lại thay vì ghi đè. Todo status và Canteen pay/cancel dùng tư duy này.

### 8.16. Timeout, retry và circuit breaker khác nhau?

Timeout giới hạn thời gian chờ; retry thử lại lỗi tạm thời nhưng có thể nhân tải và chỉ an toàn với thao tác idempotent; circuit breaker tạm ngừng gọi dependency lỗi để hệ thống hồi phục. Code hiện có HTTP timeout và Rabbit retry, **chưa có bằng chứng circuit breaker active**, nên không nhận đã triển khai.

### 8.17. Readiness khác liveness?

Liveness: process có nên được restart không. Readiness: instance đã sẵn sàng nhận traffic/dependency thiết yếu có dùng được không. Gộp mọi dependency vào liveness có thể tạo restart loop khi DB ngoài bị lỗi.

### 8.18. Docker image và container?

Image là artifact bất biến; container là instance chạy của image với config/runtime state. CD build image đúng SHA, VPS load image và recreate container; volume giữ Redis/Rabbit data. Không lưu secret vào image.

### 8.19. Tại sao healthcheck rollback chưa đủ?

Healthcheck chỉ chứng minh endpoint cơ bản trong thời gian chờ. Nó không phát hiện mọi regression, data corruption hay lỗi business hiếm. Cần smoke/canary, metric/error budget và migration tương thích; single VPS hiện chưa có canary thật.

### 8.20. Request ID khác trace ID?

Request ID là correlation identifier được truyền qua log/hop. Distributed trace có span, parent/child, timing và sampling theo từng operation. NRApp active có request ID JSON log, chưa có trace backend active.

### 8.21. RPS, VU, latency percentile?

- VU: số user ảo đồng thời.
- RPS: request hoàn thành/gửi mỗi giây, phụ thuộc latency và think time.
- p95: 95% request không chậm hơn giá trị đó; 5% còn lại có thể rất chậm.
- Success rate không thay thế latency SLO.
- Một workload đọc rỗng/chung account không suy ra capacity toàn sản phẩm.

### 8.22. RPO và RTO?

RPO là lượng dữ liệu tối đa chấp nhận mất theo thời gian; lịch daily hướng tới khoảng 24 giờ. RTO là thời gian phục hồi dịch vụ. NRApp đã restore Mongo test nhưng chưa đủ dữ liệu để cam kết RTO production/full stack.

### 8.23. Một request đi qua NestJS theo thứ tự nào?

Ở mức phỏng vấn có thể nhớ: middleware → guard → interceptor trước → pipe → controller/service → interceptor sau → exception filter nếu có lỗi. Trong NRApp, request ID/rate limit là middleware; JWT và role là guard; DTO validation là pipe; global exception filter chuẩn hóa lỗi. Thứ tự cụ thể còn phụ thuộc global/controller/route scope, nên không dùng interceptor để thay authorization guard.

### 8.24. Module, controller, provider/service và Dependency Injection?

- Module gom một capability và khai báo `imports/controllers/providers/exports`.
- Controller map HTTP input/output, nên mỏng.
- Service/provider giữ nghiệp vụ và được Nest container khởi tạo/inject.
- DI giúp thay implementation/mock khi test và quản lý lifecycle; tránh tự `new` dependency rải rác.

Trong source, Gateway controller nhận DTO/user rồi gọi service proxy; Auth/Canteen service giữ rule và repository/model được inject.

### 8.25. Node.js xử lý đồng thời thế nào?

JavaScript chạy callback trên event loop chính; I/O được hệ điều hành/libuv xử lý bất đồng bộ. Node phù hợp workload I/O như HTTP/DB/broker, nhưng CPU-bound dài sẽ chặn event loop. Hash bcrypt dùng native thread-pool; vẫn cần giới hạn concurrency/cost. Với xử lý ảnh/CPU nặng nên đưa sang worker/queue hoặc service riêng và đo event-loop lag.

### 8.26. Khi nào dùng `Promise.all`, khi nào không?

Dùng khi các tác vụ độc lập và chấp nhận fail-fast, ví dụ query rows và count hoặc load nhiều tập dữ liệu không phụ thuộc nhau. Không dùng để song song hóa các bước có thứ tự/invariant, hoặc tạo hàng nghìn promise không giới hạn. NRApp batch-load menu/category và query list/count song song để giảm latency; consumer/mail vẫn giới hạn bằng prefetch.

### 8.27. Bcrypt bảo vệ password thế nào?

Bcrypt là password hashing chậm có salt; cùng password vẫn cho hash khác, tăng chi phí brute force. Cost 10 trong Auth là tham số cần benchmark theo phần cứng/SLO, không phải con số đúng cho mọi hệ thống. Không mã hóa password để giải mã lại và không log password/hash.

### 8.28. Hash, HMAC và encryption khác gì?

- Hash một chiều kiểm tra fingerprint/integrity, ví dụ SHA-256 checksum archive; không có secret nên một mình nó không chứng minh nguồn.
- HMAC kết hợp secret để kiểm integrity + authenticity, dùng cho internal identity/Casso webhook.
- Encryption có thể giải mã bằng key, dùng Age để giữ bí mật nội dung backup.

Không dùng SHA-256 thuần thay password hash; password cần thuật toán chậm như bcrypt/Argon2.

### 8.29. 400, 401, 403, 404, 409, 429, 502 và 503?

- `400`: input/nghiệp vụ không hợp lệ.
- `401`: chưa xác thực/token không hợp lệ.
- `403`: đã xác thực nhưng không có quyền.
- `404`: resource không tồn tại hoặc được che giấu.
- `409`: xung đột trạng thái/race/duplicate.
- `429`: vượt rate limit.
- `502`: Gateway không nhận response hợp lệ từ upstream.
- `503`: dependency/cấu hình tạm chưa sẵn sàng.

Điều quan trọng hơn thuộc lòng là status nhất quán và không làm lộ thông tin nhạy cảm.

### 8.30. Offset pagination và cursor pagination?

Offset (`skip/limit`) dễ làm UI page-number nhưng page sâu chậm và dữ liệu có thể dịch khi insert. Cursor theo `(createdAt, _id)` ổn định/hiệu quả hơn cho feed/message lớn. Todo/Canteen list hiện dùng pagination kiểu page/limit; Chat message chưa pagination, nên hướng nâng cấp là cursor với compound index.

### 8.31. N+1 query là gì?

Một query lấy N record rồi gọi/query thêm một lần cho từng record, làm số round-trip tăng tuyến tính. Chat list enrich partner theo từng chat có nguy cơ này. Cách sửa là lấy các user ID duy nhất rồi batch một request như Todo directory enrichment, hoặc materialized snapshot nếu consistency cho phép.

### 8.32. Vì sao Payment staged dùng row lock và unique constraint?

Hai webhook/process có thể cùng xử lý một intent. Transaction với row lock serialize thay đổi trên payment row; unique provider event chặn cùng event được ghi hai lần. Application check đơn thuần vẫn race. Isolation/lock chỉ nên giữ ngắn và theo thứ tự nhất quán để giảm deadlock.

### 8.33. Unit, integration và end-to-end test khác nhau?

- Unit test cô lập rule/class, nhanh nhưng không chứng minh wiring/DB thật.
- Integration test kiểm nhiều thành phần như repository, Mongo/Rabbit/Redis hoặc contract giữa Gateway–service.
- E2E đi qua HTTP/auth/database như client thật, chậm hơn nhưng bắt lỗi cấu hình.

Với NRApp, rule state transition/signature/DTO nên unit test; outbox–consumer và conditional race cần integration; register→OTP→verify và create→pay/cancel cần E2E ở môi trường test.

### 8.34. Backpressure là gì?

Khi producer nhanh hơn consumer/dependency, phải giới hạn lượng việc đang xử lý thay vì để RAM/connection tăng vô hạn. Mail dùng Rabbit prefetch; outbox lấy batch hữu hạn; HTTP có timeout; container có memory limit. Queue vẫn cần monitor depth/age, vì “đã đưa vào queue” không có nghĩa là hệ thống theo kịp.

### 8.35. Idempotency key cho API write dùng thế nào?

Client gửi một key duy nhất cho một ý định; server lưu key cùng request fingerprint/result dưới unique constraint. Retry cùng payload trả kết quả cũ, key trùng payload khác bị reject. Hữu ích cho create order/payment khi client timeout không biết request đầu đã thành công chưa. Conditional status update hiện giúp pay/cancel idempotent một phần, nhưng create order chưa có client idempotency key hoàn chỉnh.

---

## 9. Câu hỏi thiết kế hệ thống và hướng trả lời

### 9.1. “Nếu traffic tăng gấp 10?”

Không trả lời ngay “thêm server”. Trình tự:

1. Xác định SLO, workload đọc/ghi/socket và bottleneck bằng metrics/profile.
2. Tách load generator khỏi VPS; dùng dataset và nhiều account thật hơn.
3. Scale Gateway/Chat cần chuyển rate-limit/presence sang shared Redis, Socket.IO Redis adapter và xem sticky session.
4. Giảm synchronous introspection DB read bằng cache/revocation strategy được đo lường.
5. Batch/paginate Chat/User queries, tối ưu index/query, connection pool.
6. Tách broker/DB/monitoring khỏi app host nếu tài nguyên cạnh tranh; thêm replica/load balancer khi thực sự cần.
7. Thêm alert/SLO, rolling/canary deploy và backup/DR phù hợp.

### 9.2. “Nếu RabbitMQ down lúc đăng ký?”

Credential + outbox vẫn commit. Publisher thấy broker chưa ready thì giữ event, retry sau; profile tạm chưa xuất hiện. Cần monitor outbox age/backlog và UI chịu eventual consistency. Nếu Mongo transaction commit lỗi thì cả credential và outbox rollback.

### 9.3. “Nếu event gửi hai lần?”

User consumer so event/version trong transaction. Event đã xử lý hoặc version không mới bị bỏ qua. Không dựa vào việc RabbitMQ sẽ chỉ gửi một lần.

### 9.4. “Nếu hai người cùng đặt một bàn?”

Đổi trạng thái bàn bằng atomic conditional update; chỉ một request match trạng thái cho phép. Tuy nhiên reserve table và save order là hai bước, nên failure sau bước một cần compensation. Thiết kế mạnh hơn có transaction hoặc reservation record có TTL/idempotency key.

### 9.5. “Nếu webhook thanh toán gửi lại?”

Trong Payment staged, provider event ID có unique constraint, row được lock và xử lý trong transaction. Webhook trùng trả kết quả idempotent, không cộng tiền/trạng thái hai lần. Outbox sau commit cũng cần consumer idempotent.

### 9.6. “Làm sao không lộ secret?”

Secret qua env/GitHub Secrets, SSH key per-service bị forced command, host key pin, log redaction, không log Mongo URI/token/password. Backup mã hóa Age trước khi off-site; private key giải mã không nằm trên VPS/GitHub. Cần thêm secret rotation/manager nếu lên production lớn.

---

## 10. Những điểm yếu nên chủ động thừa nhận

Một ứng viên tốt không nói hệ thống hoàn hảo. Có thể chọn 2–3 điểm liên quan câu hỏi:

1. Gateway rate limiter là in-memory; chưa dùng được cho nhiều replica.
2. Auth active chỉ có `admin/user`, trong khi domain enum có role khác; contract cần thống nhất.
3. Access token 7 ngày dài; nên rút ngắn và quản lý session/revocation tốt hơn.
4. OTP nên hash/HMAC; thêm throttling theo IP/device và chống enumeration toàn luồng.
5. Một protected request introspect Auth tạo latency/coupling; cần thiết kế cache/revocation theo SLO.
6. Chat presence in-memory, lịch sử chưa cursor pagination, tạo conversation có race và list có nguy cơ N+1.
7. Một số internal User endpoint tương thích cũ chưa HMAC chặt như directory route; cần đóng bằng network/auth thống nhất.
8. Canteen reserve bàn + save order dùng compensation, chưa transaction.
9. Payment có source nhưng chưa tích hợp/deploy active.
10. Monitoring active mới ở host metrics; chưa có app metrics/tracing/alerting hoàn chỉnh.
11. Backup chưa gồm Cloudinary/message RabbitMQ và chưa restore full stack.
12. Load test 50 VU thành công về HTTP nhưng tail latency chưa đạt; con số 3 phút cần raw evidence.

Công thức nói điểm yếu:

> “Hiện tại X vì bối cảnh Y. Rủi ro là Z. Nếu yêu cầu/traffic tăng, em sẽ đo A rồi chuyển sang B; em chưa nhận là đã triển khai B.”

---

## 11. Các câu tuyệt đối không nên nói

- “Hệ thống của em chịu được 50 người” — 50 VU chung account không phải 50 user thật/capacity.
- “26.5 RPS là nhanh” — phải gắn SLO và latency; run hiện có trượt p95/p99.
- “RabbitMQ bảo đảm exactly once.”
- “JWT an toàn nên không cần lưu/revoke gì.”
- “CORS ngăn hacker gọi API.”
- “Dùng microservices nên scale vô hạn.”
- “Docker healthcheck chứng minh production ổn.”
- “Redis là database cache cho mọi API” — không đúng code.
- “Payment/PostgreSQL/Casso đang chạy” — hiện không đúng deployment.
- “MySQL dùng trong NRApp” — không có bằng chứng source/runtime.
- “Có distributed tracing/Loki/Alertmanager” — không có trong stack active đã xác minh.
- “Backup tự động nghĩa là chắc chắn restore được mọi thứ.”

---

## 12. Cheat sheet code để mở ngay khi được hỏi

| Chủ đề                              | File                                                                                       |
| ----------------------------------- | ------------------------------------------------------------------------------------------ |
| Gateway bootstrap/CORS/Socket proxy | [gateway/src/main.ts](gateway/src/main.ts)                                                 |
| Authenticated request/introspection | [gateway/src/common/security/jwt.strategy.ts](gateway/src/common/security/jwt.strategy.ts) |
| HMAC internal identity              | [gateway signature](gateway/src/common/security/internal-request-signature.service.ts)     |
| Gateway rate limit                  | [rate-limit.middleware.ts](gateway/src/common/middleware/rate-limit.middleware.ts)         |
| Register/login/OTP/Google/refresh   | [auth.service.ts](auth/src/modules/auth/auth.service.ts)                                   |
| Transactional outbox                | [outbox.publisher.ts](auth/src/modules/outbox/outbox.publisher.ts)                         |
| Profile idempotency/version         | [profile-sync.service.ts](user/src/modules/user/profile-sync.service.ts)                   |
| Mail retry/DLQ                      | [mail RabbitMQ service](mail/src/modules/rabbitmq/rabbitmq.service.ts)                     |
| Chat REST                           | [chat.service.ts](chat/src/modules/chat/chat.service.ts)                                   |
| Chat realtime                       | [chat.gateway.ts](chat/src/modules/chat/chat.gateway.ts)                                   |
| Todo rules/race                     | [task.service.ts](todo/src/modules/task/task.service.ts)                                   |
| Schedule transaction                | [schedule.service.ts](workschedule/src/modules/schedule/schedule.service.ts)               |
| Attendance QR                       | [attendance.service.ts](workschedule/src/modules/attendance/attendance.service.ts)         |
| Canteen order                       | [order.service.ts](canteen/src/modules/order/order.service.ts)                             |
| Menu undo/redo                      | [undo-stack.ts](canteen/src/modules/menu/utils/undo-stack.ts)                              |
| Payment staged                      | [payment.service.ts](payment/src/modules/payment/payment.service.ts)                       |
| VPS services/resources              | [compose.vps.yaml](compose.vps.yaml)                                                       |
| CI/CD                               | [logger workflows](logger/.github/workflows)                                               |
| Backup                              | [nrapp-backup](../nrapp-backup/README.md)                                                  |
| Load test                           | [scripts/LOADTEST.md](scripts/LOADTEST.md)                                                 |

---

## 13. Checklist 15 phút trước phỏng vấn

- [ ] Nói trôi chảy intro 60 giây, không quá 90 giây.
- [ ] Kể đúng tám service active và Payment staged.
- [ ] Vẽ được luồng Client → Nginx → Gateway → service → DB/broker.
- [ ] Giải thích được register Outbox và consumer idempotent.
- [ ] Giải thích được OTP TTL/rate/attempt và refresh rotation.
- [ ] Phân biệt Gateway rate limit in-memory với Redis OTP limit.
- [ ] Nêu một race condition và cách code xử lý bằng conditional update.
- [ ] Nói rõ Chat scale cần Redis adapter/shared presence.
- [ ] Nói đúng backup có gì, thiếu gì, đã restore phần nào.
- [ ] Nhớ k6: 3 API đọc, shared account, 50 VU, 26.45 RPS, 100% success, latency chưa đạt; bằng chứng hiện có giữ 60 giây.
- [ ] Chuẩn bị lý do chọn đúng công ty/JD.
- [ ] Chuẩn bị lịch làm việc và khoảng lương nếu HR yêu cầu.
- [ ] Không mở `.env`, secret, token, IP quản trị hoặc dữ liệu người dùng khi share màn hình.

## 14. Câu hỏi nên hỏi lại interviewer

Chọn 2–3 câu phù hợp:

1. Trong 1–2 tháng đầu, anh/chị kỳ vọng intern/fresher tự làm được loại task nào?
2. Team review code và hướng dẫn thiết kế/database như thế nào?
3. Backend hiện có SLO, monitoring và quy trình xử lý incident ra sao?
4. Một feature đi từ ticket tới production qua những bước test/review/deploy nào?
5. Thách thức kỹ thuật lớn nhất của team trong quý này là gì?
6. Tiêu chí đánh giá sau thử việc/internship là gì?

---

## 15. Chốt dự án trong 30 giây

> NRApp là personal project giúp em đi hết vòng đời backend chứ không chỉ CRUD. Tám service active chạy trên một VPS qua Nginx/Gateway; Auth dùng password + email OTP, JWT/refresh rotation và Outbox để đồng bộ User; Chat dùng Socket.IO; Todo, lịch làm và Canteen có rule/race handling riêng. Em tự làm Docker Compose, CI/CD rollback, JSON request ID, monitoring và backup mã hóa. Em cũng biết rõ giới hạn: single host/replica, role contract còn lệch, Payment chưa active và load test success nhưng tail latency chưa đạt. Nếu vào team, em muốn biến kinh nghiệm tự học đó thành code được review và vận hành theo yêu cầu thật.
