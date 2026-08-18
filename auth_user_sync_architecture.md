# PHÂN TÍCH CHUYÊN SÂU KIẾN TRÚC ĐỒNG BỘ DỮ LIỆU: AUTH SERVICE & USER SERVICE
## (API Gateway Composition vs. Event-Driven Materialized View)

---

## I. TỔNG QUAN VỀ BÀI TOÁN HỆ THỐNG

Trong kiến trúc Microservices, việc phân chia trách nhiệm (Single Responsibility Principle) giữa **Auth Service** (quản lý credentials, JWT, xác thực security) và **User Service** (quản lý thông tin profile người dùng, avatar, bio...) là một thiết kế tiêu chuẩn.

Tuy nhiên, bài toán nảy sinh khi hệ thống cần thực hiện các tác vụ đọc dữ liệu tổng hợp (ví dụ: `GET ALL USERS` phục vụ màn hình Admin Dashboard hoặc danh sách tìm kiếm), trong đó response trả về cần chứa **cả thông tin xác thực** (`email`, `role`, `status`) lẫn **thông tin profile** (`username`, `avatar`, `phone`).

Tài liệu này phân tích chi tiết 2 giải pháp kiến trúc phổ biến, so sánh ưu/nhược điểm, phân tích bài toán lưu dư thừa dữ liệu (Data Redundancy) và khả năng mở rộng (Scalability) khi hệ thống tăng trưởng.

---

## II. LUỒNG 1: DYNAMIC API COMPOSITION AT GATEWAY (GỘP DỮ LIỆU ĐỘNG)

### 1. Cơ chế hoạt động
API Gateway đóng vai trò là Orchestrator. Khi nhận yêu cầu `GET /api/users` từ Client:
1. Gateway gửi 2 yêu cầu HTTP REST song song:
   - `GET /api/auth/internal/users` -> Lấy danh sách `{ userId, email, role }` từ Auth DB.
   - `GET /api/user/internal/users` -> Lấy danh sách `{ userId, username, avatar }` từ User DB.
2. Gateway chờ cả 2 `Promise` hoàn tất.
3. Gateway thực hiện thuật toán ghép (Join Array In-Memory) theo `userId`.
4. Gateway trả về danh sách đã gộp cho Client.

```
Client ──► API Gateway ──┬──► [REST Call] Auth Service ──► Auth DB
                         └──► [REST Call] User Service ──► User DB
             (In-Memory Join)
```

### 2. Ưu điểm
- **Thiết kế đơn giản, dễ triển khai**: Không tốn hạ tầng Message Broker (RabbitMQ/Kafka), không cần viết Consumer/Publisher code.
- **Tính nhất quán tuyệt đối (Strong Consistency)**: Dữ liệu trả về luôn là mới nhất tại thời điểm query ở cả 2 database.
- **Không dư thừa dữ liệu (Zero Data Redundancy)**: Mỗi database chỉ lưu đúng các field thuộc phạm vi của nó (`email` duy nhất ở Auth DB, `username` duy nhất ở User DB).

### 3. Nhược điểm & Rào cản khi Mở rộng (Scale Bottlenecks)

#### a) "Gãy" ở bài toán Phân trang, Tìm kiếm & Sắp xếp (Pagination, Search, Sort)
- **Kịch bản thực tế**: Client gọi `GET /api/users?search=gmail&sort=username&page=2&limit=20`.
- **Thách thức**: 
  - Điều kiện lọc `email LIKE '%gmail%'` nằm ở **Auth DB**.
  - Điều kiện sắp xếp `ORDER BY username ASC` nằm ở **User DB**.
- **Hệ quả**: Không thể áp dụng `LIMIT/OFFSET` hay `INDEX` ở tầng Database của từng service. API Gateway buộc phải **fetch toàn bộ dữ liệu** của cả 2 service về RAM Gateway, join lại, tự sort, rồi mới cắt 20 record ra.
- **Nghẽn**: Với 1.000 users thì chạy được; với 500.000 users, Gateway sẽ bị **bùng nổ dung lượng RAM** và CPU nghẽn 100%.

#### b) Sập dây chuyền (Cascading Failure & Latency Amplification)
- Latency tổng của request đọc bằng: `Max(Latency_Auth, Latency_User) + Network_Overhead + Gateway_Merge_Time`.
- Nếu Auth Service bị treo DB hoặc gián đoạn, API xem danh sách người dùng trên User Service cũng chết theo, mặc dù User Service vẫn hoạt động hoàn toàn bình thường.

#### c) Gateway bị phình to (Fat Gateway Antipattern)
- Gateway bị ép phải chứa business logic ghép nối JSON, vi phạm nguyên tắc chỉ nên đảm nhận Gateway Routing, Rate Limiting, Authentication Check.

---

## III. LUỒNG 2: EVENT-DRIVEN MATERIALIZED VIEW VIA RABBITMQ (ĐỒNG BỘ BẤT ĐỒNG BỘ)

### 1. Cơ chế hoạt động (Pattern CQRS & Materialized View)
User Service chủ động duy trì một **bản sao dữ liệu phẳng (Read Model)** bao gồm cả thông tin hiển thị (`username`, `avatar`) lẫn thông tin thuộc tính phục vụ tìm kiếm/lọc (`email`, `role`).

1. **Luồng Ghi (Write Path)**:
   - Khi có thay đổi ở Auth Service (Đăng ký, Đổi email, Đổi role, Xóa tài khoản), Auth Service ghi DB của mình rồi phát một Event (ví dụ: `USER_CREATED`, `USER_EMAIL_UPDATED`) lên **RabbitMQ queue: `user-profile-sync`**.
   - User Service (Consumer) lắng nghe queue, tiêu thụ event và cập nhật/tạo bản ghi tương ứng trong **User DB**.
2. **Luồng Đọc (Read Path)**:
   - Khi Client gọi `GET /api/users`, API Gateway **chuyển thẳng (Proxy)** request đến User Service.
   - User Service tự query 1 câu lệnh duy nhất trên User DB của mình và trả về kết quả ngay lập tức.

```
[WRITE PATH]
Auth Service ──► Auth DB ──► Publish Event ──► RabbitMQ ──► User Service Consumer ──► User DB (Update Read Model)

[READ PATH]
Client ──► API Gateway ──► User Service ──► User DB ──► Response
```

### 2. Ưu điểm

#### a) Hiệu năng Đọc cực cao (High Read Performance)
- Toàn bộ dữ liệu nằm sẵn trong 1 Database.
- Phân trang (`skip`, `limit`), sắp xếp (`sort`), tìm kiếm (`search`) được xử lý trực tiếp tại Database bằng **Index (B-Tree/Hash Index)**. 
- Thời gian phản hồi luôn ở mức cực thấp (< 10-20ms) kể cả khi dataset lên tới hàng triệu records.

#### b) Khả năng chịu lỗi tuyệt đối (Resilience / Fault Tolerance)
- Auth Service bị sập hoặc bảo trì? Client **vẫn duyệt danh sách user, tìm kiếm user bình thường** trên User Service.
- Nếu User Service bị down tạm thời, các event từ Auth Service vẫn được lưu an toàn trong durable queue của RabbitMQ. Khi User Service sống lại, nó tiêu thụ hết queue và dữ liệu lại đồng bộ mà không mất mát byte nào.

#### c) Giảm tải Network & Decoupled hoàn toàn
- API Gateway không cần thực hiện bất kỳ lệnh join/promise nào.
- 2 service không cần biết địa chỉ IP / internal REST endpoint của nhau.

### 3. Nhược điểm
- **Độ phức tạp hạ tầng**: Yêu cầu vận hành RabbitMQ cluster, cài đặt Dead Letter Queue (DLQ), retry mechanism.
- **Tính nhất quán sau cùng (Eventual Consistency)**: Có một khoảng trễ rất ngắn (thường 5-100ms) từ lúc đổi email ở Auth Service cho tới khi User Service cập nhật xong bản sao.

---

## IV. PHÂN TÍCH CHUYÊN SÂU: BÀI TOÁN DƯ THỪA DỮ LIỆU (DATA REDUNDANCY)

Một câu hỏi kiến trúc cốt lõi: *"Lưu dư thừa email, role ở cả Auth DB và User DB có bị coi là vi phạm thiết kế không?"*

### 1. Góc nhìn Chế độ Chế bản chuẩn hóa (Database Normalization) vs. Microservices Modern Design
- **Trong RDBMS đơn khối (Monolith)**: Bạn học nguyên tắc Chuẩn hóa DB (1NF, 2NF, 3NF) để tránh dư thừa dữ liệu nhằm tiết kiệm dung lượng đĩa cứng.
- **Trong Microservices & Distributed Systems**: Dung lượng lưu trữ (Storage/RAM) hiện nay **rất rẻ**, trong khi chi phí cho Network Call, Latency, và CPU overhead khi JOIN跨-service lại **cực kỳ đắt đỏ**. Vì vậy, tư duy thiết kế chuyển sang **Phản chuẩn hóa (Denormalization)** phục vụ tốc độ đọc.

### 2. Mô hình CQRS (Command Query Responsibility Segregation)
- **Auth Service = Command Side (Source of Truth)**: Quản lý các lệnh nạp/sửa đổi nhạy cảm (Đăng nhập, Hash Password, Đổi mật khẩu, Xác thực OTP). Chỉ có Auth Service có quyền phát hành event làm thay đổi dữ liệu gốc.
- **User Service = Query Side (Read Model / Materialized View)**: Quản lý góc nhìn hiển thị và tìm kiếm. User Service **không sở hữu bản gốc của email/role**, mà chỉ lưu **bản sao chỉ đọc (Read Replica)** để phục vụ truy vấn tốc độ cao.

---

## V. BẢNG SO SÁNH TOÀN DIỆN CÁC TIÊU CHÍ

| Tiêu chí so sánh | Luồng 1: Gateway Composition (REST) | Luồng 2: Event-Driven (RabbitMQ / Materialized View) |
| :--- | :--- | :--- |
| **Cơ chế kết hợp data** | Ghép mảng trong RAM của Gateway ở Runtime | Đã được đồng bộ sẵn trong DB của User Service |
| **Tốc độ truy vấn Đọc** | Chậm (Chờ 2 REST requests + Join time) | **Cực nhanh** (1 Query duy nhất trên 1 DB) |
| **Phân trang / Sort / Filter** | **Gãy khi data lớn** (Phải load full data về Gateway) | **Tối ưu tuyệt đối** (Dùng DB Indexing, Skip/Limit) |
| **Độ chịu lỗi (Fault Tolerance)** | Kém (1 trong 2 service lỗi = Request thất bại) | **Cực cao** (User Service chạy độc lập khi xem data) |
| **Tài nguyên Gateway** | Tốn nhiều CPU & RAM (Merge JSON payloads) | Rất nhẹ (Chỉ đơn thuần là Reverse Proxy) |
| **Tính nhất quán dữ liệu** | Strong Consistency (Nhất quán tức thì) | Eventual Consistency (Nhất quán sau vài ms) |
| **Hạ tầng hỗ trợ** | Rất đơn giản (Không cần Broker) | Phức tạp hơn (Cần RabbitMQ, Monitor Queues, DLQ) |
| **Khả năng Scale (Scale-out)** | Khó mở rộng khi dataset > 50.000 items | Dễ dàng mở rộng lên hàng triệu items |

---

## VI. CÁC NGUYÊN TẮC BẮT BUỘC ĐỂ LUỒNG 2 ĐẠT CHUẨN PRODUCTION

Để triển khai Luồng 2 (Event-Driven) một cách tin cậy trong môi trường doanh nghiệp lớn, hệ thống cần đáp ứng 2 kỹ thuật bổ trợ:

### 1. Tính kháng trùng lặp (Idempotent Consumer)
- Do cơ chế của RabbitMQ là `At-least-once delivery` (Có thể gửi lại tin nhắn nhiều lần nếu có sự cố nghẽn mạng), Consumer ở User Service bắt buộc phải có tính kháng trùng.
- **Cách làm**: Khi nhận event `UPDATE_EMAIL` với `userId`, Consumer kiểm tra nếu bản ghi đã có thông tin mới hoặc kiểm tra `eventSequenceId`, nếu đã xử lý rồi thì bỏ qua (nạp idempotent key vào Redis/DB).

### 2. Pattern Mẫu ghi Outbox (Transactional Outbox Pattern)
- Tránh rủi ro: Auth Service đã `COMMIT` transaction update DB thành công, nhưng ứng dụng bị crash ngay trước khi kịp push message lên RabbitMQ -> Khiến dữ liệu 2 bên lệch vĩnh viễn.
- **Cách làm**:
  1. Trong cùng 1 DB Transaction ở Auth Service: Cập nhật `Credentials` VÀ chèn 1 record event vào bảng `Outbox`.
  2. Một Background Worker (hoặc Debezium CDC) sẽ quét bảng `Outbox`, đẩy tin nhắn lên RabbitMQ và đánh dấu đã gửi.

---

## VII. KẾT LUẬN & ĐỀ XUẤT NẠP KIẾN TRÚC

1. **Khi nào dùng Luồng 1 (Composition)**:
   - Dùng cho các tác vụ lấy chi tiết 1 đối tượng duy nhất theo ID (ví dụ: `GET /api/users/:id`). Việc gọi 2 service lấy detail 1 user rồi ghép JSON lại rất nhẹ và không vướng bài toán phân trang.
   - Các dự án MVP, vừa và nhỏ, traffic thấp, cần ship nhanh.

2. **Khi nào bắt buộc dùng Luồng 2 (Event-Driven Materialized View)**:
   - Tất cả các màn hình danh sách, tìm kiếm, phân trang, lọc nhiều điều kiện (`GET ALL USERS`).
   - Các hệ thống định hướng Microservices sẵn sàng mở rộng (Scalable Production Architecture).
