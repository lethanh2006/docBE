# Tài liệu hướng dẫn kiến trúc và source code - Chat Service

Tài liệu này giải thích chi tiết cấu trúc thư mục `src` và logic hoạt động của Backend **Chat Service** (dịch vụ trò chuyện) trong hệ thống microservices.

## 1. Tổng quan cấu trúc thư mục `src`

Thư mục `src` được tổ chức theo mô hình MVC (Model-View-Controller) kết hợp với các cấu hình (config) và middleware:

```
src/
├── config/         # Cấu hình kết nối DB, Socket.IO, Cloudinary, Error Handler
├── controllers/    # Xử lý logic nghiệp vụ cho các API Chat
├── middleware/     # Các middleware trung gian (Xác thực, Upload file)
├── models/         # Định nghĩa các schema MongoDB (Mongoose)
├── routes/         # Định nghĩa các route API
└── index.ts        # File entry point (khởi tạo app, server)
```

---

## 2. Chi tiết các thành phần

### 2.1. File `index.ts` (Entry Point)
- Khởi tạo ứng dụng **Express** và tích hợp **HTTP Server** cùng **Socket.IO** (từ `config/socket.ts`).
- Kết nối tới MongoDB thông qua `connectDb()`.
- Thiết lập CORS, JSON body parser.
- Khởi tạo API Health check (`/health`).
- Gắn (mount) các route chat tại đường dẫn `/api/chat`.
- Cấu hình Global Error Handler (đặc biệt xử lý lỗi file size của Multer).
- Tích hợp **Swagger** (`swagger-jsdoc`, `swagger-ui-express`) để tạo tài liệu API tự động tại `/api-docs`.

### 2.2. Thư mục `config/`
Chứa các thiết lập cốt lõi của ứng dụng.

- **`db.ts`**: Kết nối với MongoDB sử dụng Mongoose. Có set cứng DNS Server (`8.8.8.8`) và chỉ định DB Name là `nrapp`.
- **`socket.ts`**: Thiết lập WebSocket thông qua thư viện **Socket.IO** để phục vụ Real-time Chat.
  - Sử dụng Middleware để xác thực token JWT khi client kết nối tới Socket.
  - Dùng một `Map` (`userSocketMap`) lưu trữ mapping giữa `userId` và tập hợp các `socketId` (hỗ trợ 1 user đăng nhập trên nhiều thiết bị).
  - Lắng nghe và xử lý các sự kiện (Events): 
    - `typing` / `typingStop`: Gửi trạng thái đang gõ phím tới người nhận.
    - `disconnect`: Xóa socket của user khi họ thoát.
    - Phát (emit) sự kiện `getOnlineUsers` cho tất cả client để hiển thị ai đang online.
- **`cloudinary.ts`**: Cấu hình thông tin xác thực cho Cloudinary (lưu trữ ảnh) lấy từ biến môi trường.
- **`TryCatch.ts`**: Hàm bọc (Wrapper function) giúp bắt lỗi (try-catch) tự động cho các hàm controller bất đồng bộ (async), giúp code gọn gàng, không cần lặp lại `try...catch` nhiều lần.

### 2.3. Thư mục `models/`
Định nghĩa cấu trúc dữ liệu lưu trong MongoDB.

- **`Chat.ts`**: Schema đại diện cho một cuộc hội thoại.
  - `users`: Mảng chứa danh sách ID người tham gia (string).
  - `latestMessage`: Chứa nội dung và người gửi của tin nhắn cuối cùng (dùng để hiển thị ở danh sách chat).
- **`Messages.ts`**: Schema đại diện cho một tin nhắn cụ thể.
  - `chatId`: Tham chiếu đến ID của cuộc hội thoại (Chat).
  - `sender`: ID của người gửi.
  - `text`: Nội dung tin nhắn văn bản.
  - `image`: Object chứa URL và Public ID của ảnh trên Cloudinary.
  - `messageType`: Phân loại tin nhắn ("text" hoặc "image").
  - `seen` / `seenAt`: Trạng thái đã xem và thời gian xem.

### 2.4. Thư mục `middleware/`
- **`isAuth.ts`**: Middleware bảo vệ các API.
  - Hỗ trợ mô hình API Gateway: Ưu tiên đọc payload user từ header `x-user-payload` (đã được Gateway giải mã và truyền vào dạng Base64).
  - Hỗ trợ gọi trực tiếp: Đọc và giải mã JWT token từ header `Authorization` (Bearer token).
  - Bóc tách lấy thông tin user và gắn vào `req.user` cho các middleware/controller phía sau xử lý.
- **`multer.ts`**: Cấu hình upload file với thư viện Multer và `CloudinaryStorage`.
  - Giới hạn file chỉ là ảnh (jpg, jpeg, png, gif) và dung lượng tối đa 5MB.
  - Ảnh tự động được resize/crop về kích thước 800x800 trước khi lưu lên Cloudinary folder `chat-images`.

### 2.5. Thư mục `controllers/`
Chứa toàn bộ logic xử lý API trong `chat.ts`:

- **`createNewChat`**: Tạo một phòng chat mới giữa user hiện tại và một `otherUserId`. Hệ thống có cơ chế gọi qua microservice `USER_SERVICE` bằng Axios để kiểm tra người dùng `otherUserId` có tồn tại hay không trước khi tạo.
- **`getAllChats`**: Lấy danh sách tất cả các cuộc trò chuyện của user hiện tại. Nó gọi sang `USER_SERVICE` để lấy thông tin profile của đối phương và đồng thời đếm số lượng tin nhắn chưa đọc (`unseenCount`).
- **`sendMessage`**: Xử lý việc gửi tin nhắn.
  - Nhận text hoặc file ảnh (đã được upload bởi multer).
  - Lưu tin nhắn vào DB `Messages` và cập nhật `latestMessage` của collection `Chat`.
  - Gửi event `newMessage` thông qua Socket.IO tới các `socketId` của người nhận ngay lập tức (real-time).
- **`getMessagesByChat`**: Lấy toàn bộ tin nhắn trong một cuộc hội thoại.
  - Thực hiện logic đánh dấu đã xem (`seen: true`) cho tất cả các tin nhắn gửi đến.
  - Gửi event `messagesSeen` qua Socket.IO tới người gửi tin nhắn để client hiển thị trạng thái "Đã xem".

### 2.6. Thư mục `routes/`
- **`chat.ts`**: Định nghĩa các endpoint (API path) cho Chat service:
  - `POST /api/chat/chat/new`: Tạo chat mới (cần xác thực).
  - `GET /api/chat/chat/all`: Lấy tất cả chat (cần xác thực).
  - `POST /api/chat/message`: Gửi tin nhắn, bao gồm upload 1 ảnh qua key `image` (cần xác thực).
  - `GET /api/chat/message/:chatId`: Lấy danh sách tin nhắn của 1 chat (cần xác thực).

---

## 3. Luồng hoạt động (Data Flow) nổi bật

1. **Giao tiếp liên dịch vụ (Inter-service Communication)**:
   Chat Service không giữ thông tin User (tên, avatar) trong cơ sở dữ liệu của mình mà chỉ lưu `userId`. Mỗi khi cần hiển thị danh sách chat, nó sẽ gọi HTTP GET sang `USER_SERVICE/api/user/internal/:userId` để lấy thông tin Profile.
2. **Xác thực qua API Gateway**:
   Việc xác thực được ưu tiên lấy từ header `x-user-payload` giả định rằng một hệ thống API Gateway phía trước đã xử lý verify token và nhúng thông tin user vào header, giúp giảm tải giải mã JWT liên tục ở các Service.
3. **Chat Real-time (Socket.IO)**:
   - Các sự kiện gõ tin nhắn (`typing`), tin nhắn mới (`newMessage`), đánh dấu đã xem (`messagesSeen`) được xử lý rất mượt thông qua việc query `userSocketMap` trong bộ nhớ RAM của server để tìm ra đúng danh sách các Socket ID của user nhận.
