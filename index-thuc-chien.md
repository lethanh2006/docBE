# Index Thực Chiến — 7 Loại Index Theo Mục Đích, Chạy Bên Trong Ra Sao

> Tài liệu bổ sung cho `indexDB.md`. Tổ chức lại đúng theo 7 loại index ở mục 4 tài liệu gốc
> (4.1 → 4.7). Với **mỗi loại**, đi đủ 5 phần:
> **Mục đích thực tế → SQL → Cơ chế bên trong khi chạy (từng bước) → EXPLAIN annotated → Sai lầm thường gặp.**

---

## Bảng tra nhanh 7 loại

| # | Loại Index | Dùng khi nào (mục đích) | Node xuất hiện trong EXPLAIN |
|---|---|---|---|
| 4.1 | Single-Column | Query chỉ filter đúng 1 cột | `Index Scan` |
| 4.2 | Composite | Filter nhiều cột cùng lúc, có thể kèm sort | `Index Scan`, không có `Sort` nếu đúng thứ tự |
| 4.3 | Unique | Cột cần đảm bảo không trùng ở tầng DB (email, SKU, username) | `Index Scan` + kiểm tra ràng buộc lúc ghi |
| 4.4 | Partial | Chỉ một phần nhỏ rows thực sự được query (soft-delete, pending) | `Index Scan` — index nhỏ hơn hẳn |
| 4.5 | Covering | Query chỉ cần vài cột, muốn tránh đọc heap | `Index Only Scan`, `Heap Fetches: 0` |
| 4.6 | Clustered vs Non-Clustered | Thiết kế Primary Key và ảnh hưởng đến mọi secondary index | InnoDB: double lookup; PostgreSQL: không áp dụng khái niệm clustered mặc định |
| 4.7 | Expression/Functional | Query luôn bọc hàm quanh cột (`LOWER()`, `EXTRACT()`) | `Index Scan` chỉ khi query dùng ĐÚNG expression |

---

## 4.1 Single-Column Index

### Mục đích thực tế
Query chỉ bao giờ filter đúng **1 cột**, không kết hợp với cột khác. Ví dụ: trang admin tra cứu user theo email, hoặc filter đơn hàng theo `created_at`.

### SQL
```sql
CREATE INDEX idx_users_email ON users (email);

SELECT * FROM users WHERE email = 'alice@gmail.com';
```

### Cơ chế bên trong khi chạy
1. Optimizer thấy điều kiện `=` khớp đúng cột đầu của 1 index sẵn có → chọn `Index Scan`.
2. Traversal B-Tree: Root → Internal → Leaf (thường chỉ 3–4 lần đọc page dù bảng có hàng triệu row, vì cây rất "phẳng" — độ sâu tăng logarit theo số row).
3. Tại Leaf tìm đúng entry → lấy con trỏ `(page, row)`.
4. Nhảy sang heap đọc đúng page đó, lấy toàn bộ row (vì `SELECT *`).

### EXPLAIN annotated
```
Index Scan using idx_users_email on users
  (cost=0.42..8.44 rows=1 width=120)
  Index Cond: (email = 'alice@gmail.com')
```
`cost=0.42` = chi phí traversal B-Tree gần như bằng 0; `rows=1` = optimizer biết trước cardinality email rất cao nên ước tính đúng chỉ 1 row.

### Sai lầm thường gặp
- Tạo single-column index cho **cột có cardinality thấp** (`gender`, `is_active`) — optimizer thường bỏ qua, chọn Seq Scan vì đọc index + heap tốn hơn đọc thẳng bảng khi >~15-20% rows match.
- Tạo single-column index rồi *sau đó* thêm composite index bao gồm luôn cột đó ở vị trí đầu → single-column trở thành **redundant** (xem Anti-pattern 3 trong tài liệu gốc), cần drop.

---

## 4.2 Composite Index (Multi-Column)

### Mục đích thực tế
Query chạy **rất thường xuyên** (API endpoint nóng), luôn filter nhiều cột cùng lúc.

### SQL
```sql
CREATE INDEX idx_orders_user_status_time
  ON orders (userId, status, created_at DESC);

SELECT * FROM orders
WHERE userId = 123 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 20;
```

### Cơ chế bên trong khi chạy
1. Traversal tới leaf đầu tiên thoả `userId=123 AND status='pending'` — vì `userId` là cột equality đầu tiên, index thu hẹp phạm vi cực nhanh.
2. Trong phạm vi đó, các entry **đã được sắp sẵn theo `created_at DESC`** (vì đây là cột thứ 3 trong định nghĩa index) → không cần bước sắp xếp riêng.
3. Đọc tuần tự 20 entry đầu → dừng ngay nhờ `LIMIT`, không cần duyệt hết.

### EXPLAIN annotated
```
Limit
  ->  Index Scan using idx_orders_user_status_time on orders
        Index Cond: ((userId = 123) AND (status = 'pending'))
-- Không có node "Sort" — dấu hiệu index đang phát huy tối đa.
```

Nếu tạo sai thứ tự cột (vd `(status, userId, created_at)` trong khi 90% query luôn có `userId` trước) hoặc thiếu `created_at` trong index, plan sẽ hiện thêm:
```
Sort
  Sort Key: created_at DESC
  ->  Index Scan using idx_orders_user_status on orders
```
→ Sort tốn thêm CPU + có thể tràn ra disk nếu tập kết quả lớn.

### Sai lầm thường gặp
- Vi phạm **Leftmost Prefix Rule**: index `(A,B,C)` không dùng được cho query chỉ có `WHERE B = ?` — phải quét toàn bộ index hoặc bỏ qua hoàn toàn.
- Sắp cột range (`>`, `<`, `BETWEEN`) **trước** cột equality — làm mất khả năng "thu hẹp" sớm. Thứ tự đúng luôn là: equality → range → sort.

---

## 4.3 Unique Index

### Mục đích thực tế
Không chỉ để tăng tốc — mục đích chính là **ép ràng buộc không trùng lặp ở tầng database**, để dù application layer có bug (race condition, thiếu validate) thì DB vẫn chặn được duplicate. Ví dụ: email đăng ký, SKU sản phẩm, username.

### SQL
```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users (email);
-- hoặc:
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);
```

### Cơ chế bên trong khi chạy
Với **SELECT**, cơ chế giống hệt B-Tree thường (4.1) — không có gì khác biệt về tốc độ đọc.

Điểm khác nằm ở **INSERT/UPDATE**:
1. Trước khi ghi entry mới vào leaf, engine phải **traversal tìm xem key đó đã tồn tại chưa** (giống một lần Index Scan ẩn, xảy ra trước khi ghi thật).
2. PostgreSQL dùng khoá ở mức index page ngắn hạn để tránh 2 transaction cùng insert trùng giá trị đồng thời (tránh race condition).
3. Nếu trùng → transaction bị abort với lỗi `duplicate key value violates unique constraint`.
4. Nếu không trùng → ghi entry mới bình thường (như B-Tree insert thường, có thể gây page split).

### EXPLAIN annotated
Với SELECT, plan giống hệt single-column B-Tree. Điểm cần quan sát thực chiến nằm ở **hành vi lúc INSERT dưới tải cao**: 2 transaction cùng insert 1 email gần như đồng thời — transaction thứ 2 sẽ bị chặn ngắn hoặc abort ngay khi phát hiện trùng, chứ không đợi tới lúc commit mới báo lỗi.

### Sai lầm thường gặp
- Nghĩ unique index "miễn phí" — thực ra **mỗi INSERT tốn thêm 1 lần traversal kiểm tra trùng**, đắt hơn B-Tree thường một chút.
- Chỉ validate unique **ở application layer** (check-rồi-mới-insert) mà không có Unique Index ở DB → dễ dính race condition: 2 request kiểm tra gần như đồng thời đều thấy "chưa tồn tại", rồi cùng insert thành công → dữ liệu trùng lặp âm thầm mà app không phát hiện.

---

## 4.4 Partial Index (Filtered Index)

### Mục đích thực tế
Bảng lớn nhưng **query thực tế chỉ quan tâm một tập con nhỏ** — điển hình nhất là soft-delete (`WHERE deleted_at IS NULL`) hoặc trạng thái đang xử lý (`WHERE status = 'pending'`) trong khi phần lớn rows đã ở trạng thái kết thúc.

### SQL
```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC LIMIT 20;
```

### Cơ chế bên trong khi chạy
1. Vì index **chỉ chứa entry của rows thoả `status='pending'`** (giả sử 50K/10M rows), cây B-Tree của nó **nông hơn và nhỏ hơn rất nhiều** so với index full-table cùng cột.
2. Toàn bộ index có thể **vừa gọn trong RAM buffer pool**, trong khi index full-table 10M rows có thể phải đọc từ disk.
3. Traversal diễn ra y hệt B-Tree thường, nhưng số page phải đọc ít hơn hẳn → nhanh hơn đáng kể, đồng thời **không tốn overhead ghi** khi INSERT/UPDATE rows có `status != 'pending'` (engine không cần cập nhật index này cho những rows đó).

### EXPLAIN annotated
```
Index Scan using idx_orders_pending on orders
  (cost=0.15..120.5 rows=50000 width=64)
  -- so sánh với index full-table cùng cột:
  -- (cost=0.43..185000.00 rows=50000 width=64) nếu phải quét toàn bộ 10M-row index
```
Điểm quan trọng: optimizer **chỉ dùng được Partial Index nếu điều kiện WHERE của query "bao hàm" (implies) điều kiện của index**. Nếu query viết `WHERE status IN ('pending', 'processing')`, PostgreSQL **không tự suy luận** được là có overlap với `WHERE status = 'pending'` trong định nghĩa index → có thể bỏ qua index này hoàn toàn.

### Sai lầm thường gặp
- Viết điều kiện WHERE trong query **không khớp chính xác** biểu thức trong Partial Index (kể cả khi về mặt logic tương đương) → optimizer không nhận ra và bỏ qua index.
- Dùng Partial Index cho điều kiện mà tỉ lệ rows thoả mãn **thay đổi nhiều theo thời gian** (hôm nay 5% pending, 6 tháng sau tăng lên 40%) — lợi ích "nhỏ gọn" giảm dần, cần review định kỳ.

---

## 4.5 Covering Index (Index-Only Scan)

### Mục đích thực tế
Query chạy **cực kỳ thường xuyên** (endpoint nóng nhất hệ thống) và chỉ cần vài cột cụ thể, không cần `SELECT *`. Mục tiêu: loại bỏ hoàn toàn bước đọc heap.

### SQL
```sql
CREATE INDEX idx_orders_cover ON orders (userId) INCLUDE (status, total);

SELECT status, total FROM orders WHERE userId = 123;
```

### Cơ chế bên trong khi chạy
1. Traversal tới leaf như B-Tree thường.
2. Vì `status, total` đã nằm sẵn trong leaf (nhờ `INCLUDE`) → về lý thuyết không cần đọc heap nữa.
3. **Chi tiết PostgreSQL hay bị bỏ sót:** trước khi bỏ qua heap, engine phải kiểm tra **Visibility Map** của heap page tương ứng — cấu trúc theo dõi page nào "all-visible" (không còn dead tuple chưa dọn) do MVCC.
   - Page đã được VACUUM sạch → visibility map báo "visible" → thực sự bỏ qua heap → `Index Only Scan` đúng nghĩa, `Heap Fetches: 0`.
   - Page chưa VACUUM gần đây → vẫn phải đọc heap để double-check tính hợp lệ của row → bạn tưởng được tối ưu nhưng thực ra vẫn tốn I/O như cũ.

### EXPLAIN annotated
```
-- Case tốt:
Index Only Scan using idx_orders_cover on orders
  Index Cond: (userId = 123)
  Heap Fetches: 0

-- Case chưa tối ưu (bảng UPDATE nhiều, chưa vacuum):
Index Only Scan using idx_orders_cover on orders
  Index Cond: (userId = 123)
  Heap Fetches: 847        <- vẫn đang đọc heap ngầm, cần VACUUM
```

### Sai lầm thường gặp
- Tạo Covering Index cho bảng **UPDATE liên tục** mà không kèm chiến lược VACUUM hợp lý → `Heap Fetches` luôn cao, index chiếm thêm storage/RAM mà không đạt lợi ích thật sự.
- Nhồi quá nhiều cột vào `INCLUDE` "cho chắc" → index phình to, làm chậm INSERT/UPDATE ở mọi cột được include (dù chúng không nằm trong điều kiện WHERE, index vẫn phải cập nhật giá trị của chúng mỗi lần thay đổi).

---

## 4.6 Clustered vs Non-Clustered Index

### Mục đích thực tế
Đây không phải loại index bạn "chọn tạo" trực tiếp mà là **quyết định kiến trúc** ảnh hưởng đến mọi index khác trên bảng — đặc biệt quan trọng khi thiết kế Primary Key cho hệ thống dùng MySQL/InnoDB.

### SQL
```sql
-- MySQL InnoDB: PRIMARY KEY LUÔN LÀ clustered index, không cần khai báo riêng
CREATE TABLE orders (
    id      BIGINT AUTO_INCREMENT PRIMARY KEY,  -- clustered
    userId INT,
    status  VARCHAR(20)
);
CREATE INDEX idx_orders_status ON orders (status);  -- non-clustered (secondary)

SELECT * FROM orders WHERE status = 'paid';
```

### Cơ chế bên trong khi chạy — điểm khác biệt lớn nhất giữa MySQL và PostgreSQL

**MySQL InnoDB:**
1. Data rows được **sắp xếp vật lý trên disk theo Primary Key** — clustered index chính là bảng.
2. Tra `idx_orders_status` (non-clustered/secondary) → nhưng leaf của nó **không chứa row pointer trực tiếp**, mà chứa **giá trị Primary Key**.
3. Lấy PK tìm được → tra tiếp vào clustered index (PK B-Tree) để lấy toàn bộ row.
4. → Đây là **2 lần traversal B-Tree** ("double lookup"/"bookmark lookup"), thay vì 1.

```
EXPLAIN kỳ vọng (MySQL):
type: ref | key: idx_orders_status | Extra: (không có "Using index")
-- "Using index" xuất hiện khi query chỉ cần cột đã có trong secondary index
-- (tức là secondary index đó đã "covering", tránh được lần lookup thứ 2)
```

**PostgreSQL:** không có khái niệm clustered mặc định. Mọi index (kể cả Primary Key) đều là non-clustered — leaf chứa CTID trỏ thẳng vào heap, chỉ 1 lần lookup duy nhất (như mục 4.1). Bạn có thể chạy `CLUSTER` **một lần** để sắp xếp lại heap theo 1 index cụ thể, nhưng đây là thao tác thủ công, không tự duy trì khi có INSERT mới.

### Sai lầm thường gặp
- Dùng **UUID string làm Primary Key trong MySQL InnoDB** → mọi secondary index đều phải lưu UUID dài trong leaf (do double lookup cần PK) → index phình to, chậm cả insert lẫn lookup, và giá trị UUID random còn gây **page split liên tục** vì insert không theo thứ tự tăng dần (khác hẳn AUTO_INCREMENT luôn insert vào cuối).
- Mang tư duy "PK nên là UUID cho MySQL" áp dụng y hệt sang PostgreSQL mà không cân nhắc — ảnh hưởng ở PostgreSQL nhẹ hơn nhiều vì cơ chế lookup khác, nhưng UUID vẫn tốn thêm 16 bytes/entry so với BIGINT ở mọi nơi PK được tham chiếu (FK, index khác).

---

## 4.7 Expression / Functional Index

### Mục đích thực tế
Query của bạn **luôn luôn** bọc một hàm quanh cột trước khi so sánh — case-insensitive search, trích xuất năm từ timestamp, chuẩn hoá số điện thoại... Index thường trên cột gốc **sẽ không được dùng** trong các trường hợp này.

### SQL
```sql
CREATE INDEX idx_users_email_lower ON users (LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'alice@gmail.com';  -- dùng index
SELECT * FROM users WHERE email = 'alice@gmail.com';         -- KHÔNG dùng index này
```

### Cơ chế bên trong khi chạy
1. Khi `CREATE INDEX`, engine **không lưu giá trị gốc của `email`** trong leaf — nó tính trước `LOWER(email)` cho từng row hiện có và lưu **kết quả đó** làm key.
2. Khi query chạy, optimizer so khớp **chuỗi biểu thức trong WHERE** với **chuỗi biểu thức đã định nghĩa trong index** — phải khớp gần như văn bản (cùng hàm, cùng tham số, cùng kiểu dữ liệu).
3. Nếu khớp → traversal B-Tree bình thường như mục 4.1, chỉ khác key là `lower('alice@gmail.com')` thay vì giá trị gốc.
4. Nếu **không khớp biểu thức** (ví dụ viết `WHERE email ILIKE 'alice@gmail.com'` thay vì `LOWER(email) = ...`) → optimizer coi như không có index nào phù hợp → Seq Scan toàn bảng.

### EXPLAIN annotated
```
-- Query đúng biểu thức:
Index Scan using idx_users_email_lower on users
  Index Cond: (lower(email) = 'alice@gmail.com')

-- Query sai biểu thức (dùng email trực tiếp):
Seq Scan on users
  Filter: (lower(email) = 'alice@gmail.com')   <- PostgreSQL vẫn tính LOWER() ở filter,
                                                   nhưng phải tính cho TỪNG ROW sau khi
                                                   đã đọc toàn bộ bảng, mất hết lợi ích index
```

### Sai lầm thường gặp
- Đội dev viết không thống nhất: chỗ này dùng `LOWER(email) = ?`, chỗ khác dùng `email ILIKE ?`, chỗ khác nữa dùng `citext` — mỗi cách cần loại index khác nhau, Expression Index chỉ cứu được đúng 1 pattern viết trùng khớp.
- Tạo Expression Index cho hàm **không immutable** (kết quả có thể thay đổi theo thời gian/session, ví dụ hàm phụ thuộc timezone hiện tại) — PostgreSQL sẽ từ chối tạo index hoặc dữ liệu trong index có thể sai lệch dần.
- Quên rằng Expression Index **tốn chi phí tính toán lại ở mỗi INSERT/UPDATE** (phải chạy hàm để tính key mới) — đắt hơn B-Tree thường một chút, cần cân nhắc nếu bảng ghi rất nhiều.

---

## Ghi chú quan trọng: Bitmap trong EXPLAIN không phải "Bitmap Index" ở mục 3.3

Một điểm hay gây nhầm khi đọc EXPLAIN thực tế: bạn có thể thấy node `Bitmap Index Scan` / `Bitmap Heap Scan` xuất hiện dù **không hề tạo loại index nào ở trên bằng bitmap**. Đó là vì khi optimizer cần **kết hợp 2 index B-Tree khác nhau** cho cùng 1 query (ví dụ đã có `idx_orders_status` và `idx_orders_city` riêng biệt — hai Single-Column Index ở mục 4.1 — cho query lọc cả 2 cột), nó tự dựng **bitmap tạm trong RAM** từ mỗi B-Tree, AND/OR bitwise lại với nhau, rồi mới đọc heap theo thứ tự vật lý (tối ưu I/O). Đây khác hẳn Bitmap Index kiểu Oracle (lưu trữ vĩnh viễn, mục 3.3 tài liệu gốc) — PostgreSQL không có loại index lưu trữ dạng đó, nhưng **cơ chế bitwise AND/OR vẫn hoạt động ngầm** mỗi khi bạn để nhiều Single-Column Index cho optimizer tự kết hợp thay vì gộp cứng vào 1 Composite Index.

```
EXPLAIN ANALYZE WHERE status = 'pending' AND city = 'HCM'
(đã có idx_orders_status và idx_orders_city riêng, KHÔNG có composite):

Bitmap Heap Scan on orders
  Recheck Cond: ((status = 'pending') AND (city = 'HCM'))
  ->  BitmapAnd
        ->  Bitmap Index Scan on idx_orders_status
        ->  Bitmap Index Scan on idx_orders_city
```

→ Bài học: nếu thấy `BitmapAnd`, đó là tín hiệu optimizer đang kết hợp tốt nhiều Single-Column Index (4.1) — bạn **không bắt buộc** phải gộp thành Composite Index (4.2) trừ khi query đó chạy đủ thường xuyên để việc tối ưu thêm (bỏ bước AND) tạo ra khác biệt đo được.

---

## Quy trình chọn 1 trong 7 loại cho 1 query cụ thể

```
1. Query filter đúng 1 cột, không cần đảm bảo unique, không phải tập con nhỏ?
   → 4.1 Single-Column

2. Query filter nhiều cột CÙNG LÚC và chạy rất thường xuyên?
   → 4.2 Composite (nhớ: equality trước, range giữa, sort cột cuối)

3. Cột này về mặt nghiệp vụ KHÔNG được trùng (email, SKU, username)?
   → 4.3 Unique — tạo dù có validate ở app layer hay không

4. Phần lớn rows KHÔNG BAO GIỜ được query tới (soft-delete, đã archive)?
   → 4.4 Partial — chỉ index tập con thực sự cần

5. Query này là "hot path", chỉ cần vài cột cụ thể, không cần SELECT *?
   → 4.5 Covering — nhớ kèm chiến lược VACUUM để Heap Fetches = 0

6. Đang thiết kế Primary Key cho MySQL/InnoDB?
   → 4.6 Cân nhắc Clustered — ưu tiên BIGINT AUTO_INCREMENT, tránh UUID làm PK

7. Query LUÔN bọc hàm quanh cột trước khi so sánh (LOWER, EXTRACT, chuẩn hoá)?
   → 4.7 Expression — nhưng phải đảm bảo MỌI query dùng đúng 1 biểu thức thống nhất
```

---

*Đọc song song với `indexDB.md` mục 4 (định nghĩa) và mục 10 (EXPLAIN) — tài liệu này chỉ thêm phần "chạy thật ra sao", không thay thế lý thuyết gốc.*
