# 📋 Todo Service — Tài liệu dự án

> Microservice quản lý công việc (Task Management) trong hệ thống NRApp.  
> Port mặc định: **5003** | Database: **MongoDB Atlas** | DB name: `nrapp`

---

## 📁 Cấu trúc thư mục

```
todo/
├── src/
│   ├── index.ts              # Entry point — khởi động server Express
│   ├── config/
│   │   └── db.ts             # Kết nối MongoDB
│   ├── controllers/
│   │   └── task.ts           # Xử lý logic nghiệp vụ
│   ├── middleware/
│   │   └── isAuth.ts         # Xác thực người dùng qua header
│   ├── model/
│   │   └── Task.ts           # Schema Mongoose cho Task
│   └── routes/
│       └── task.ts           # Định nghĩa API routes + Swagger docs
├── dist/                     # Output sau khi build TypeScript
├── .env                      # Biến môi trường
├── package.json
└── tsconfig.json
```

---

## 🏗️ Kiến trúc tổng quan

```
[API Gateway / Client]
        │
        │  Header: x-user-payload (base64 JSON)
        ▼
[Todo Service :5003]
   ├── isAuth Middleware     ← giải mã user từ header
   ├── Routes /api/todo/...  ← định tuyến request
   ├── Controllers           ← xử lý logic, phân quyền
   ├── Model (Task)          ← tương tác MongoDB
   └── User Service (HTTP)   ← kiểm tra user tồn tại, lấy danh sách user
```

> **Lưu ý:** Service này **không tự xác thực JWT**. Việc xác thực do API Gateway thực hiện, sau đó truyền thông tin user qua header `x-user-payload` (base64 encoded JSON).

---

## ⚙️ Biến môi trường (`.env`)

| Biến | Giá trị mặc định | Mô tả |
|------|------------------|-------|
| `MONGO_URL` | `mongodb+srv://...` | Connection string MongoDB Atlas |
| `PORT` | `5003` | Cổng lắng nghe của service |
| `REDIS_URL` | `redis://127.0.0.1:6379` | (Hiện chưa dùng trong code) |
| `Rabbitmq_Host` | `localhost` | (Hiện chưa dùng trong code) |
| `Rabbitmq_Username` | `guest` | (Hiện chưa dùng trong code) |
| `Rabbitmq_Password` | `guest` | (Hiện chưa dùng trong code) |
| `JWT_SECRET` | `f0ee52e...` | (Hiện chưa dùng trực tiếp — JWT do Gateway xử lý) |
| `USER_SERVICE_URL` | `http://localhost:5000` | URL của User Service (fallback) |

---

## 📦 Model: `Task`

**File:** `src/model/Task.ts`

| Trường | Kiểu | Bắt buộc | Mặc định | Mô tả |
|--------|------|----------|----------|-------|
| `title` | `String` | ✅ | — | Tiêu đề công việc |
| `description` | `String` | ❌ | — | Mô tả chi tiết |
| `status` | `Enum` | ❌ | `todo` | Trạng thái hiện tại |
| `priority` | `Enum` | ❌ | `medium` | Mức độ ưu tiên |
| `createdBy` | `ObjectId` | ✅ | — | ID người tạo (ref: User) |
| `assignedTo` | `ObjectId` | ❌ | — | ID người được giao (ref: User) |
| `deadline` | `Date` | ❌ | — | Hạn chót hoàn thành |
| `createdAt` | `Date` | auto | — | Thời gian tạo (timestamps) |
| `updatedAt` | `Date` | auto | — | Thời gian cập nhật (timestamps) |

### Các giá trị Enum

**`status`:**
- `todo` — Chưa bắt đầu *(mặc định)*
- `in_progress` — Đang thực hiện
- `done` — Đã hoàn thành
- `cancelled` — Đã huỷ

**`priority`:**
- `low` — Thấp
- `medium` — Trung bình *(mặc định)*
- `high` — Cao

---

## 🔐 Middleware: `isAuth`

**File:** `src/middleware/isAuth.ts`

Middleware này **không xác thực JWT trực tiếp**. Thay vào đó, nó đọc header `x-user-payload` (do API Gateway inject vào sau khi xác thực JWT), giải mã base64 và parse JSON để lấy thông tin user.

```
Request Header: x-user-payload: <base64(JSON.stringify(user))>
                                        │
                                        ▼
                          { _id, username, email, role, ... }
                                        │
                                        ▼
                               req.user = user
```

Nếu header thiếu hoặc payload không hợp lệ → trả về `401 Unauthorized`.

---

## 🛣️ API Endpoints

**Base URL:** `/api/todo`  
**Swagger UI:** `http://localhost:5003/api-docs`  
**Swagger JSON:** `http://localhost:5003/api/docs.json`

### Tất cả routes đều yêu cầu xác thực (`isAuth`)

| Method | Endpoint | Quyền | Mô tả |
|--------|----------|-------|-------|
| `GET` | `/api/todo/my-tasks` | User / Admin | Lấy danh sách công việc được giao cho mình |
| `PATCH` | `/api/todo/:id/status` | User được giao / Admin | Cập nhật trạng thái công việc |
| `POST` | `/api/todo/` | **Admin only** | Tạo công việc mới |
| `PATCH` | `/api/todo/:id/assign` | **Admin only** | Giao lại công việc cho người khác |
| `GET` | `/api/todo/` | **Admin only** | Lấy tất cả công việc trong hệ thống |
| `DELETE` | `/api/todo/:id` | **Admin only** | Xoá công việc |
| `GET` | `/health` | Public | Health check |

---

## 🧠 Controller: `task.ts` — Chi tiết xử lý

**File:** `src/controllers/task.ts`

### Helper functions

| Hàm | Mô tả |
|-----|-------|
| `isAdmin(req)` | Kiểm tra role === `"admin"` (case-insensitive) |
| `rejectNonAdmin(req, res)` | Từ chối nếu không phải admin, trả 403 |
| `isExistingUser(userId)` | Gọi HTTP đến User Service để kiểm tra user tồn tại |
| `isValidTaskId(id, res)` | Kiểm tra ObjectId hợp lệ |
| `populateUsersInTasks(tasks, payload)` | Gọi User Service để lấy thông tin user (username, email) và nhúng vào danh sách task |
| `getUserServiceUrl()` | Đọc URL của User Service từ env |

### Luồng xử lý các API chính

#### `POST /` — Tạo công việc
```
1. Kiểm tra Admin
2. Validate title (không rỗng)
3. Nếu có assignedTo → gọi User Service kiểm tra user tồn tại
4. Tạo Task mới với createdBy = req.user._id
5. Lưu DB → trả 201
```

#### `PATCH /:id/assign` — Giao lại công việc
```
1. Kiểm tra Admin
2. Validate ObjectId
3. Tìm task theo id
4. Gọi User Service kiểm tra người được giao tồn tại
5. Cập nhật assignedTo → lưu DB → trả 200
```

#### `GET /` — Lấy tất cả task (Admin)
```
1. Kiểm tra Admin
2. Task.find().sort({ createdAt: -1 })
3. populateUsersInTasks → gọi User Service lấy thông tin users
4. Trả danh sách task đã được populate
```

#### `GET /my-tasks` — Lấy task của tôi
```
1. Không cần Admin
2. Task.find({ assignedTo: req.user._id })
3. populateUsersInTasks
4. Trả danh sách
```

#### `PATCH /:id/status` — Cập nhật trạng thái
```
1. Validate ObjectId + status hợp lệ
2. Tìm task theo id
3. Kiểm tra quyền: phải là người được giao HOẶC Admin
4. Cập nhật status → lưu → trả 200
```

#### `DELETE /:id` — Xoá task
```
1. Kiểm tra Admin
2. Validate ObjectId
3. findByIdAndDelete → trả 200 (hoặc 404 nếu không tìm thấy)
```

---

## 🔗 Tích hợp với User Service

Todo Service giao tiếp với **User Service** qua HTTP thuần (không dùng message queue):

| Mục đích | Endpoint gọi đến User Service |
|----------|-------------------------------|
| Kiểm tra user tồn tại | `GET /api/user/internal/:userId` |
| Lấy danh sách tất cả users (để populate) | `GET /api/user/user/all` |

URL của User Service được cấu hình qua `USER_SERVICE_URL` hoặc `USER_SERVICE` trong `.env`, fallback về `http://localhost:5000`.

---

## 🚀 Cách chạy dự án

### Cài đặt dependencies
```bash
cd backend/todo
npm install
```

### Chạy môi trường development
```bash
npm run dev
# Chạy song song: tsc -w (watch TypeScript) + nodemon dist/index.js
```

### Build production
```bash
npm run build   # Compile TypeScript → dist/
npm start       # Chạy node dist/index.js
```

### Kiểm tra service
```bash
curl http://localhost:5003/health
# → { "status": "ok", "service": "todo" }
```

---

## 📚 Tech Stack

| Công nghệ | Phiên bản | Mục đích |
|-----------|-----------|----------|
| **Node.js** | — | Runtime |
| **TypeScript** | ^5.7.3 | Ngôn ngữ chính |
| **Express** | ^4.21.2 | HTTP framework |
| **Mongoose** | ^8.9.5 | ODM cho MongoDB |
| **MongoDB Atlas** | — | Database chính |
| **swagger-jsdoc** | ^6.2.8 | Tự động sinh OpenAPI spec từ JSDoc |
| **swagger-ui-express** | ^5.0.1 | Serve Swagger UI |
| **cors** | ^2.8.5 | Cross-Origin Resource Sharing |
| **dotenv** | ^16.4.5 | Đọc biến môi trường |
| **concurrently** | ^9.1.2 | Chạy song song tsc + nodemon |
| **nodemon** | ^3.1.9 | Tự động reload khi file thay đổi |

---

## ⚠️ Lưu ý quan trọng

> [!WARNING]
> Các biến `REDIS_URL`, `Rabbitmq_*`, và `JWT_SECRET` có trong `.env` nhưng **hiện chưa được sử dụng** trong source code. Đây có thể là chuẩn bị cho tính năng tương lai hoặc còn sót từ template.

> [!NOTE]
> Service này hoạt động theo mô hình **microservice với Gateway pattern**. Toàn bộ xác thực JWT được xử lý bên ngoài (API Gateway), Todo Service chỉ tin tưởng vào header `x-user-payload` đã được inject. Điều này có nghĩa là service **không nên expose trực tiếp** ra internet mà phải đi qua Gateway.

> [!TIP]
> Khi phát triển local mà không có Gateway, bạn có thể test bằng cách tự tạo header `x-user-payload`:
> ```js
> const payload = Buffer.from(JSON.stringify({ _id: "...", role: "admin" })).toString("base64");
> // Gửi header: x-user-payload: <payload>
> ```
