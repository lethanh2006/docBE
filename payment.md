# Payment Service — tài liệu chuẩn

> Trạng thái: đã triển khai. Cập nhật lần cuối: 20/08/2026.

Tài liệu này là điểm vào chính thức cho Payment Service của hệ thống. Đặc tả
"nạp ví game qua gRPC" trước đây đã được loại bỏ vì không khớp với domain và
source backend hiện tại. Payment hiện phục vụ thanh toán `Order` của Canteen.

## Tài liệu triển khai đầy đủ

Toàn bộ hướng dẫn kiến trúc, mã nguồn, biến môi trường, migration, API, chữ ký
HMAC, webhook Casso, outbox RabbitMQ, consumer Canteen, test và vận hành nằm tại:

- [`backend/payment/PAYMENT_SERVICE_GUIDE.md`](../backend/payment/PAYMENT_SERVICE_GUIDE.md)

Đây là nguồn tài liệu chi tiết duy nhất cần dùng khi phát triển hoặc vận hành
Payment. File `payservice.md` chỉ còn là ghi chú lưu trữ của kế hoạch ban đầu.

## Phạm vi nghiệp vụ

Payment Service nhận yêu cầu tạo QR cho một `Order` đã tồn tại trong Canteen,
theo dõi giao dịch chuyển khoản và phát sự kiện khi thanh toán được xác nhận.

Các nguyên tắc bắt buộc:

- Client chỉ gửi `orderId`, không được tự quyết định số tiền.
- Gateway lấy `finalAmount` chính thức từ Canteen rồi ký toàn bộ ngữ cảnh tạo QR.
- Payment dùng PostgreSQL; không dùng MongoDB hay Redis.
- Mỗi Order chỉ có tối đa một Payment `PENDING` và một Payment `SUCCESS`.
- Webhook được xác thực bằng HMAC-SHA512 và được chống xử lý lặp vĩnh viễn trong
  PostgreSQL.
- Cập nhật Payment và ghi outbox event diễn ra trong cùng một transaction.
- Canteen chỉ đổi `paymentStatus` thành `PAID`; không ghi `status = PAID`.
- Bàn chỉ được giải phóng khi Order vừa `COMPLETED` vừa đã thanh toán.

## Luồng end-to-end

```text
Client
  -> Gateway: POST /api/payment/create-qr { orderId }
  -> Canteen: GET order và lấy finalAmount/owner/payment state
  -> Payment: request nội bộ đã ký gồm orderId + finalAmount + identity
  -> PostgreSQL: tạo hoặc tái sử dụng Payment PENDING
  <- Client: paymentId, amount, reference, qrUrl, expiresAt

Ngân hàng -> Casso -> Gateway -> Payment webhook
  -> kiểm chữ ký, tài khoản nhận, reference, số tiền và transaction id
  -> PostgreSQL transaction: Payment SUCCESS + webhook receipt + outbox event
  -> RabbitMQ: payment.succeeded
  -> Canteen consumer: paymentStatus = PAID
```

## API công khai qua Gateway

| Method | Path | Xác thực | Mục đích |
|---|---|---|---|
| `POST` | `/api/payment/create-qr` | JWT | Tạo hoặc lấy lại QR đang chờ cho Order |
| `GET` | `/api/payment/:paymentId` | JWT | Xem trạng thái một Payment |
| `GET` | `/api/payment/order/:orderId` | JWT | Xem lịch sử Payment của Order |
| `POST` | `/api/payment/webhooks/casso` | HMAC Casso | Nhận giao dịch Casso |

Payment còn cung cấp trực tiếp:

- `GET /health/live`: tiến trình còn sống.
- `GET /health/ready`: PostgreSQL và RabbitMQ đã sẵn sàng.

## Dữ liệu và nhất quán

Tiền VND được lưu dưới dạng số nguyên, không dùng số thực. Các bảng chính:

- `payments`: payment intent và trạng thái giao dịch.
- `webhook_receipts`: idempotency vĩnh viễn theo giao dịch nhà cung cấp.
- `outbox_events`: bảo đảm event không mất khi RabbitMQ tạm ngừng.

Migration được chạy bằng TypeORM; production luôn dùng
`synchronize: false`. Không sửa schema production bằng cơ chế tự đồng bộ.

## Trạng thái

```text
PENDING -> SUCCESS
PENDING -> EXPIRED
PENDING -> REVIEW_REQUIRED
```

`REVIEW_REQUIRED` được dùng khi giao dịch cần kiểm tra thủ công, ví dụ sai số
tiền hoặc sai tài khoản nhận. Một Payment đã `SUCCESS` không được quay lại trạng
thái trước đó.

## Event tích hợp Canteen

Routing key/event type: `payment.succeeded`.

Queue của Canteen: `canteen.payment.succeeded.v1`.

Payload phiên bản 1 gồm `eventId`, `paymentId`, `orderId`, `userId`, số tiền,
đơn vị tiền, phương thức, mã giao dịch nhà cung cấp và thời điểm thanh toán.
Consumer phải idempotent theo dữ liệu Order hiện tại và chỉ ack sau khi cập nhật
thành công.

## Những thiết kế cũ không còn hiệu lực

Không triển khai các nội dung sau từ đặc tả trước đây:

- nạp tiền vào ví người chơi;
- `updateMoney`, `createFinanceRecord` hoặc lookup username;
- giao tiếp gRPC cho Payment;
- MongoDB/Mongoose trong Payment;
- Redis TTL để chống webhook lặp;
- nhận `amount` trực tiếp từ client;
- đổi lifecycle của Order thành `PAID` hoặc giải phóng bàn ngay khi nhận tiền.

Nếu sau này hệ thống cần thêm domain ví, hãy thiết kế một Wallet Service và sổ
cái riêng; không ghép số dư ví vào luồng thanh toán Order hiện tại.

## Bắt đầu nhanh

Từ thư mục `backend`:

```bash
docker compose up -d --build --wait payment-postgres rabbitmq payment canteen gateway
docker compose exec payment npm run migration:show:compiled
curl http://127.0.0.1:5006/health/ready
```

Đọc hướng dẫn đầy đủ trước khi cấu hình secret hoặc webhook production:
[`PAYMENT_SERVICE_GUIDE.md`](../backend/payment/PAYMENT_SERVICE_GUIDE.md).
