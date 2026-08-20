# Kế hoạch Payment Service cũ — đã lưu trữ

> Tài liệu này không còn là đặc tả triển khai.

Kế hoạch ban đầu trong file này dùng MongoDB, nhận số tiền từ client và giải
phóng bàn ngay khi thanh toán. Các lựa chọn đó đã được thay thế vì không bảo đảm
nguồn tiền chính xác và không đúng lifecycle của Order.

Tài liệu chuẩn hiện tại:

- [Tổng quan Payment](./payment.md)
- [Hướng dẫn triển khai và vận hành đầy đủ](../backend/payment/PAYMENT_SERVICE_GUIDE.md)

Source of truth là code trong `backend/payment`, Gateway Payment module,
Canteen payment consumer và migration PostgreSQL đi kèm.
