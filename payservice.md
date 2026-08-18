# Implementation Plan - Decoupled Payment Microservice Architecture

Tách rời tính năng thanh toán từ một module trong `Canteen Service` thành một **Payment Microservice** độc lập. Điều này giúp hệ thống có thể scale độc lập phần thanh toán khi lượng transaction tăng cao mà không cần scale toàn bộ nghiệp vụ Canteen (gọi món, nhà bếp, kho bãi).

## User Review Required

> [!IMPORTANT]
> **Quy trình kết nối qua Event-Driven (RabbitMQ)**
> 1. Client gọi API sinh mã QR thanh toán qua Gateway tới **Payment Service**: `POST /api/payment/create-qr` (truyền `orderId`, `amount`).
> 2. Payment Service lưu giao dịch trạng thái `PENDING` vào Database của nó, trả về link/QR.
> 3. Khi thanh toán thành công (Webhook callback từ ngân hàng/ví điện tử gọi tới `POST /api/payment/callback`), Payment Service cập nhật trạng thái giao dịch thành `SUCCESS`, đồng thời bắn sự kiện `payment.succeeded` lên RabbitMQ.
> 4. Canteen Service lắng nghe event `payment.succeeded` để tự động cập nhật trạng thái Order tương ứng sang `PAID` và cập nhật bàn ăn sang `empty`.
>
> Hãy xác nhận luồng Event-driven này đã đúng ý anh/chị chưa.

## Proposed Changes

### 1. Cấu hình định tuyến Gateway

#### [MODIFY] [index.js](file:///d:/Chatapp/backend/gateway/src/index.js)
- Thêm cấu hình định tuyến cho `/api/payment` chuyển tiếp tới Payment Service (`http://localhost:5006`).
- Thêm `PAYMENT_SERVICE_URL` vào biến môi trường của Gateway.

---

### 2. Canteen Service (Dọn dẹp Payment Module)

#### [DELETE] [payment.module.ts](file:///d:/Chatapp/backend/canteen/src/modules/payment/payment.module.ts)
- Xóa bỏ module thanh toán cũ trong Canteen Service.

#### [MODIFY] [app.module.ts](file:///d:/Chatapp/backend/canteen/src/app.module.ts)
- Bỏ import `PaymentModule`.

#### [NEW] [payment.consumer.ts](file:///d:/Chatapp/backend/canteen/src/modules/order/consumers/payment.consumer.ts)
- Viết consumer trong Order Module lắng nghe event `payment.succeeded` từ RabbitMQ để cập nhật trạng thái đơn hàng sang `PAID` và cập nhật bàn ăn sang trống.

---

### 3. Tạo mới Payment Service (`d:\Chatapp\backend\payment`)

#### [NEW] Khởi tạo dự án Payment Service [NEW PROJECT]
- Khởi tạo dự án NestJS mới tại thư mục `d:\Chatapp\backend\payment`.
- Sao chép cấu hình `.env`, `tsconfig.json`, `nest-cli.json` tương tự như `canteen` service.
- Sử dụng Port mặc định: `5006`.
- Kết nối tới Database riêng: `chatapp_payment` để tránh dùng chung DB với Canteen.

#### [NEW] [payment.schema.ts](file:///d:/Chatapp/backend/payment/src/schemas/payment.schema.ts)
Lưu trữ thông tin lịch sử giao dịch thanh toán:
```typescript
{
  _id: ObjectId,
  orderId: String,        // ID đơn hàng liên kết bên Canteen
  amount: Number,         // Số tiền thanh toán
  status: String,         // "PENDING" | "SUCCESS" | "FAILED" | "REFUNDED"
  transactionId: String,  // Mã giao dịch từ ví/ngân hàng
  paymentMethod: String,  // "VIETQR" | "MOMO" | "VNPAY"
  createdAt: Date
}
```

#### [NEW] [payment.service.ts](file:///d:/Chatapp/backend/payment/src/modules/payment/payment.service.ts)
- `createQrCode(orderId, amount)`: Sinh VietQR/Link thanh toán.
- `handleCallback(callbackData)`: Xử lý webhook từ nhà cung cấp, cập nhật DB thành `SUCCESS` và publish event `payment.succeeded` qua RabbitMQ.

#### [NEW] [payment.controller.ts](file:///d:/Chatapp/backend/payment/src/modules/payment/payment.controller.ts)
- Endpoint `POST /api/payment/create-qr`
- Endpoint `POST /api/payment/callback` (Webhook công khai)

---

## Verification Plan

### Automated Tests
- Viết test kiểm tra sinh mã VietQR và kiểm tra webhook callback của Payment Service.

### Manual Verification
1. Chạy Payment Service trên port `5006`.
2. Tạo đơn hàng bên Canteen (được `orderId` và `finalAmount`).
3. Gửi request sinh QR tới `/api/payment/create-qr` qua Gateway.
4. Gửi request mô phỏng callback của cổng thanh toán tới `/api/payment/callback`.
5. Xác minh qua log xem Payment Service có bắn event `payment.succeeded` sang RabbitMQ không.
6. Kiểm tra xem Canteen Service nhận được event này và tự động chuyển trạng thái đơn hàng sang `PAID` chưa.
