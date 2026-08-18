# Tài liệu Chi tiết Luồng Định tuyến và Vai trò của JwtStrategy tại API Gateway

Tài liệu này giải thích chi tiết hai khía cạnh cốt lõi của API Gateway:
1. **Vai trò và cơ chế hoạt động của `JwtStrategy`**.
2. **Cách thức khớp nối đường dẫn (Routing Map) giữa API Gateway và các Microservices con (ví dụ: Workschedule Service)**.

---

## Phần 1: Tác dụng và Cơ chế của `JwtStrategy`

Tệp tin [jwt.strategy.ts](file:///d:/Chatapp/backend/gateway/src/modules/auth/common/guard/jwt/jwt.strategy.ts) là một phần của thư viện Passport JWT được tích hợp vào NestJS Guards. Nó giải quyết bài toán: **Làm sao để biết Token này hợp lệ và ai là người gửi?**

```typescript
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt-1gio') {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), // 1. Nơi lấy Token
      ignoreExpiration: false,                                  // 2. Không bỏ qua hạn dùng
      secretOrKey: process.env.JWT_SECRET || 'your-secret',    // 3. Khóa giải mã
    });
  }

  async validate(payload: any) {
    return {
      _id: payload.userId || payload._id || payload.id,
      userId: payload.userId || payload._id || payload.id,
      username: payload.username,
      role: payload.role,
    };
  }
}
```

### Cách thức hoạt động từng bước:
1. **Trích xuất Token (Extract Token):** Khi request đi qua `JwtAuthGuard`, Passport tự động tìm header `Authorization: Bearer <Token>` để lấy chuỗi Token.
2. **Kiểm tra Chữ ký & Hạn dùng (Verify Signature & Expiration):** Passport dùng khóa bí mật `JWT_SECRET` để kiểm tra:
   * Chữ ký của Token có bị thay đổi/giả mạo không?
   * Token còn hạn sử dụng không (so với trường `exp` bên trong)?
   * *Nếu không hợp lệ, hệ thống tự động ném ra lỗi `401 Unauthorized` ngay lập tức.*
3. **Giải mã & Khớp thông tin (Validate Payload):** 
   * Khi xác thực thành công, hàm `validate(payload)` được gọi. Tham số `payload` chính là nội dung JSON đã giải mã từ Token.
   * Kết quả trả về của hàm này (Object chứa `userId`, `username`, `role`) sẽ được **NestJS tự động gán vào Request Object dưới dạng `req.user`**.
   * Nhờ đó, ở các Guards tiếp theo (`RolesGuard`) hoặc trong Controller, bạn chỉ cần gọi `req.user` là lấy được ngay thông tin người dùng hiện tại mà không cần giải mã lại.

---

## Phần 2: Luồng Khớp nối Đường dẫn (Gateway Routing Map)

### Câu hỏi đặt ra: 
> *Ví dụ API `router.get('/attendance/my', isAuth, getMyAttendance);` thì ở Gateway đường dẫn phải khớp thế nào với Swagger để chạy được?*

### 1. Khớp nối ở phía Microservice Con (Workschedule Service - Cổng 5004)
Trong tệp [workschedule/src/index.ts](file:///d:/Chatapp/backend/workschedule/src/index.ts), bạn khai báo tiền tố dịch vụ:
```typescript
app.use("/api/workschedule", attendanceRoutes);
```
Và trong tệp [routes/attendance.ts](file:///d:/Chatapp/backend/workschedule/src/routes/attendance.ts) định nghĩa route con:
```typescript
router.get('/attendance/my', isAuth, getMyAttendance);
```
👉 Đường dẫn thực tế mà Workschedule Service lắng nghe là:
`GET http://localhost:5004/api/workschedule/attendance/my`

---

### 2. Khớp nối ở phía API Gateway (Cổng 3000)
Trên API Gateway, để làm cầu nối trung gian, ta phải khai báo **đúng cấu trúc đường dẫn đó** thì Client mới gọi được và Swagger mới hiển thị chuẩn xác:

* **Tầng Controller của Gateway:**
  ```typescript
  @Controller('api/workschedule') // Định nghĩa tiền tố lớp
  @ApiTags('Api Workschedule')
  export class WorkscheduleController {
  
    @Get('attendance/my') // Khớp đúng đường dẫn con
    @ApiOperation({ summary: 'Lấy danh sách điểm danh cá nhân' })
    async getMyAttendance(@Req() req: any) {
      return this.workscheduleService.getMyAttendance(req.user);
    }
  }
  ```
  👉 Gateway phơi bày ra API: `GET http://localhost:3000/api/workschedule/attendance/my`.
  Vì được viết tường minh bằng các decorator `@Get()`, `@Controller()`, **Swagger trên Gateway sẽ tự động phát hiện và vẽ ra API này trên giao diện tài liệu**.

* **Tầng Service của Gateway (WorkscheduleService):**
  Khi Client gọi đến Gateway, Gateway chuyển tiếp chính xác request này sang Microservice:
  ```typescript
  async getMyAttendance(user: any) {
    return this.forward('GET', '/api/workschedule/attendance/my', null, null, user);
  }
  ```
  * `this.forward` sẽ gửi HTTP request đến: `${process.env.WORKSCHEDULE_SERVICE_URL}/api/workschedule/attendance/my`.

---

## Sơ đồ luồng đi của API:

```
[Client App]
     │ (Gọi: GET http://localhost:3000/api/workschedule/attendance/my)
     ▼
[API Gateway (Cổng 3000)]
     │
     ├─► 1. Đi qua JwtAuthGuard -> Kích hoạt JwtStrategy (Giải mã Token lấy user)
     ├─► 2. Đi qua RolesGuard (Không check vì API này không khai báo @Roles)
     ├─► 3. Đi vào Controller handler: getMyAttendance()
     │
     ▼ (Gọi tiếp REST API qua HttpService)
[Workschedule Service (Cổng 5004)]
     │ (Nhận: GET http://localhost:5004/api/workschedule/attendance/my)
     │ (Header: x-user-payload: <Base64 của user>)
     │
     ├─► 1. Đi qua middleware isAuth (Giải mã Base64 sang req.user)
     ├─► 2. Đi vào Controller của Service con xử lý database
     │
     ▼
[Kết quả JSON trả ngược lại cho Client]
```

### Kết luận:
1. **Có, bắt buộc phải trùng khớp.** Đường dẫn khai báo trong Controller của Gateway (`@Controller` + `@Get/Post`) phải trùng khớp với URL nghiệp vụ mà microservice con thiết lập để khi forward đi, request tìm đúng đích.
2. Việc khai báo trùng khớp và tường minh này tại Gateway giúp **Swagger tự động tạo tài liệu chính xác 100%**, đồng thời cho phép Gateway đóng vai trò gác cổng kiểm tra an ninh trước khi chuyển tiếp dữ liệu.
