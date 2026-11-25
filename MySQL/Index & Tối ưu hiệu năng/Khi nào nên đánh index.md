# Khi nào nên đánh Index?

## Nguyên tắc tổng quát: Index luôn có 2 mặt - tăng tốc độ đọc, giảm tốc độ ghi

Index giúp `SELECT` nhanh hơn nhưng làm `INSERT`, `UPDATE`, `DELETE` **chậm hơn**, vì mỗi lần ghi MySQL phải cập nhật cả bảng lẫn mọi index liên quan. Index còn **tốn dung lượng** đĩa và bộ nhớ. Vì vậy không phải cột nào cũng nên đánh index. Chỉ đánh khi **lợi ích khi đọc lớn hơn chi phí khi ghi**, tức là cột được truy vấn thường xuyên trên bảng đủ lớn.

## Khi nào nên đánh Index

**1. Cột hay xuất hiện trong `WHERE`**, đặc biệt trên bảng lớn.

```sql
CREATE INDEX idx_orders_status ON orders(status);
SELECT * FROM orders WHERE status = 'PENDING';
```

**2. Cột khóa ngoại và cột dùng trong `JOIN ... ON`.** Thiếu index ở đây thì mỗi lần `JOIN` phải quét cả bảng con.

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
SELECT u.full_name, o.id
FROM users u
JOIN orders o ON o.user_id = u.id;
```

**3. Cột hay dùng trong `ORDER BY`, `GROUP BY`.** Vì index đã được sắp xếp sẵn nên MySQL khỏi phải sắp xếp lại.

```sql
CREATE INDEX idx_orders_date ON orders(order_date);
SELECT * FROM orders ORDER BY order_date DESC LIMIT 20;
```

**4. Cột có độ phân biệt cao (nhiều giá trị khác nhau)**: email, số điện thoại, mã đơn hàng. Index loại bỏ được phần lớn dòng chỉ sau một bước.

**5. Nhiều cột thường được lọc cùng nhau**: dùng **composite index** (index nhiều cột). Thứ tự cột quan trọng: đặt cột lọc bằng `=` trước, cột khoảng hoặc sắp xếp sau.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
SELECT * FROM orders WHERE user_id = 10 AND status = 'PAID';
```

**6. Muốn tránh quay lại bảng**: index chứa đủ các cột cần lấy (covering index) cho những truy vấn chạy rất thường xuyên.

**Cách xác định cột cần index trong thực tế:**
- Dùng `EXPLAIN` xem truy vấn có `type = ALL` (quét toàn bảng) hay không.
- Bật **slow query log** để tìm những câu truy vấn chậm đang chạy thật.
- Đánh index theo **truy vấn thực tế**, không đoán trước.

## Khi nào không nên đánh Index

**1. Bảng nhỏ** (vài trăm đến vài nghìn dòng). Quét toàn bảng đã đủ nhanh, thậm chí MySQL bỏ qua index.

**2. Cột có độ phân biệt thấp** như giới tính, `is_active`, trạng thái chỉ có 2-3 giá trị. Một giá trị chiếm 50% bảng thì dùng index cũng phải đọc nửa bảng, MySQL thường chọn quét toàn bảng cho nhanh hơn.

**3. Cột hiếm khi xuất hiện trong `WHERE`, `JOIN`, `ORDER BY`.** Index không dùng đến chỉ tốn chỗ và làm ghi chậm.

**4. Bảng ghi nhiều, đọc ít** như bảng log, bảng lịch sử, bảng nhận dữ liệu liên tục. Nên giữ số index ở mức tối thiểu.

**5. Cột bị bọc trong hàm hoặc phép tính khi truy vấn**: index trên cột đó vô dụng.

```sql
-- Index trên order_date không được dùng
SELECT * FROM orders WHERE YEAR(order_date) = 2026;

-- Viết lại để dùng được index
SELECT * FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01';
```

Tương tự với `LIKE '%abc'` (dấu `%` ở đầu) hoặc so sánh sai kiểu dữ liệu.

**6. Cột chứa dữ liệu quá dài** như `TEXT`, nội dung bài viết. Index rất lớn và chậm. Nếu cần tìm kiếm văn bản thì dùng `FULLTEXT` index hoặc công cụ như Elasticsearch.

**7. Index trùng lặp hoặc thừa**: ví dụ đã có index `(user_id, status)` thì index riêng `(user_id)` là thừa, vì index đầu đã phục vụ được cả hai.

```sql
-- Kiểm tra các index đang có của một bảng
SHOW INDEX FROM orders;
```

> **Mẹo:** Mỗi bảng chỉ nên có số index vừa phải. Quá nhiều index (hơn khoảng 5-6 cái trên bảng ghi nhiều) thường là dấu hiệu cần xem lại.

## Lưu ý: UNIQUE và PRIMARY KEY tự động có index, không cần tạo thêm

- **`PRIMARY KEY`** tự tạo clustered index.
- **`UNIQUE`** tự tạo một unique index, vì MySQL cần index để kiểm tra trùng nhanh.
- **`FOREIGN KEY`**: InnoDB tự tạo index cho cột khóa ngoại nếu chưa có index phù hợp.

Vì vậy **không tạo thêm index cho các cột này**, nếu không sẽ bị trùng lặp.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,      -- đã có index
    email VARCHAR(255) NOT NULL UNIQUE,        -- đã có index
    full_name VARCHAR(100)
);

-- Thừa, không cần
CREATE INDEX idx_users_email ON users(email);
```

## Tổng kết

| Nên đánh index | Không nên đánh index |
|---|---|
| Cột trong `WHERE`, `JOIN`, `ORDER BY` trên bảng lớn | Bảng nhỏ |
| Khóa ngoại | Cột ít giá trị khác nhau (giới tính, `is_active`) |
| Cột có độ phân biệt cao (email, mã đơn) | Cột ít khi truy vấn |
| Nhóm cột hay lọc cùng nhau (composite) | Bảng ghi nhiều, đọc ít |
| Truy vấn chậm đã đo được bằng `EXPLAIN` | Cột `TEXT` dài, cột bị bọc trong hàm |

**Cách nhớ nhanh:** Đánh index cho cột **được tìm kiếm thường xuyên và có nhiều giá trị khác nhau**. Đừng đánh theo cảm tính, hãy **đo bằng `EXPLAIN` trước**.
