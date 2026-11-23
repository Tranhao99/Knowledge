# UNION

> **UNION** dùng để **gộp kết quả của 2 hay nhiều câu `SELECT`** thành một bảng kết quả duy nhất, bằng cách **xếp các dòng nối tiếp nhau** (theo chiều dọc). Khác với JOIN là nối thêm **cột** (theo chiều ngang), UNION nối thêm **dòng**.

Các ví dụ dùng bảng `customers` (`id`, `name`, `email`, `city`), `suppliers` (`id`, `name`, `email`, `city`), `orders` (`id`, `user_id`, `status`, `total_amount`, `order_date`) và `orders_archive` (cùng cấu trúc với `orders`, chứa đơn hàng cũ).

## 1. Cú pháp cơ bản

```sql
SELECT cột_1, cột_2 FROM bảng_A
UNION
SELECT cột_1, cột_2 FROM bảng_B;
```

**Ví dụ:** Danh sách email của cả khách hàng và nhà cung cấp.

```sql
SELECT name, email
FROM customers
UNION
SELECT name, email
FROM suppliers;
```

## 2. UNION và UNION ALL

| Tiêu chí | UNION | UNION ALL |
|---|---|---|
| Dòng trùng nhau | **Loại bỏ**, chỉ giữ 1 dòng | **Giữ nguyên** tất cả |
| Tốc độ | Chậm hơn (phải so sánh để loại trùng) | Nhanh hơn |
| Dùng khi | Cần danh sách không trùng | Biết chắc không trùng, hoặc cần giữ trùng |

**Ví dụ minh họa:** Nếu `a@gmail.com` có ở cả 2 bảng.

```sql
-- UNION: a@gmail.com chỉ xuất hiện 1 lần
SELECT email FROM customers
UNION
SELECT email FROM suppliers;

-- UNION ALL: a@gmail.com xuất hiện 2 lần
SELECT email FROM customers
UNION ALL
SELECT email FROM suppliers;
```

> **Lưu ý:** Dòng được coi là trùng khi **tất cả các cột** đều giống nhau. Hai dòng chỉ khác nhau ở 1 cột thì vẫn là 2 dòng khác nhau.
>
> **Mẹo hiệu năng:** Mặc định nên dùng `UNION ALL`. Chỉ dùng `UNION` khi thật sự cần loại trùng.

## 3. Quy tắc bắt buộc khi dùng UNION

- **Cùng số cột:** Các `SELECT` phải trả về số cột bằng nhau.
- **Kiểu dữ liệu tương thích:** Các cột ở cùng vị trí phải cùng kiểu hoặc chuyển đổi được (chuỗi với chuỗi, số với số).
- **Ghép cột theo vị trí, không theo tên:** Cột thứ 1 ghép với cột thứ 1, cột thứ 2 với cột thứ 2.
- **Tên cột kết quả lấy theo `SELECT` đầu tiên.**

```sql
-- Sai: SELECT đầu có 2 cột, SELECT sau có 3 cột
SELECT name, email FROM customers
UNION
SELECT name, email, city FROM suppliers;

-- Đúng
SELECT name, email FROM customers
UNION
SELECT name, email FROM suppliers;
```

> **Cẩn thận ghép nhầm vị trí:** Nếu viết `SELECT name, email` ở trên và `SELECT email, name` ở dưới, MySQL vẫn chạy được nhưng dữ liệu bị **lẫn lộn** (email nằm trong cột name). Luôn kiểm tra thứ tự cột.

## 4. ORDER BY và LIMIT với UNION

`ORDER BY` và `LIMIT` đặt **ở cuối cùng**, áp dụng cho **toàn bộ kết quả** sau khi gộp. Sắp xếp theo tên cột của `SELECT` đầu tiên.

```sql
SELECT name, email FROM customers
UNION
SELECT name, email FROM suppliers
ORDER BY name
LIMIT 10;
```

Muốn sắp xếp hoặc giới hạn **riêng cho từng phần**, bọc từng `SELECT` trong ngoặc đơn.

```sql
(SELECT name, price FROM products ORDER BY price DESC LIMIT 3)
UNION ALL
(SELECT name, price FROM products ORDER BY price ASC LIMIT 3);
```

Câu trên lấy 3 sản phẩm đắt nhất và 3 sản phẩm rẻ nhất trong cùng một kết quả.

## 5. Gắn nhãn nguồn dữ liệu

Khi gộp nhiều bảng, nên thêm **một cột hằng số** để biết mỗi dòng đến từ đâu.

```sql
SELECT name, email, 'Khách hàng' AS source
FROM customers
UNION ALL
SELECT name, email, 'Nhà cung cấp' AS source
FROM suppliers;
```

| name | email | source |
|---|---|---|
| Nguyễn An | an@gmail.com | Khách hàng |
| Công ty ABC | abc@corp.vn | Nhà cung cấp |

Cách này cũng giúp `UNION` không loại nhầm dòng khi một người vừa là khách vừa là nhà cung cấp (vì cột `source` khác nhau).

## 6. Use case phổ biến trong ứng dụng bán hàng

### a) Gộp bảng hiện tại và bảng lưu trữ (archive)

Đơn hàng cũ được chuyển sang `orders_archive` để bảng chính nhẹ. Khi cần xem toàn bộ lịch sử thì gộp lại.

```sql
SELECT id, user_id, total_amount, order_date
FROM orders
WHERE user_id = 10
UNION ALL
SELECT id, user_id, total_amount, order_date
FROM orders_archive
WHERE user_id = 10
ORDER BY order_date DESC;
```

### b) Gộp dữ liệu từ nhiều nguồn có cấu trúc khác nhau thành danh sách chung

Bảng thông báo gồm đơn hàng mới và đánh giá mới.

```sql
SELECT id, 'Đơn hàng mới' AS type, order_date AS created_at
FROM orders
WHERE status = 'PENDING'
UNION ALL
SELECT id, 'Đánh giá mới' AS type, created_at
FROM reviews
WHERE is_read = FALSE
ORDER BY created_at DESC;
```

### c) Báo cáo tổng hợp kèm dòng tổng cộng

```sql
SELECT status, COUNT(*) AS total
FROM orders
GROUP BY status
UNION ALL
SELECT 'TỔNG CỘNG', COUNT(*)
FROM orders;
```

### d) Giả lập FULL OUTER JOIN

MySQL **không có `FULL OUTER JOIN`**. Có thể tạo ra bằng `LEFT JOIN` + `RIGHT JOIN` + `UNION`:

```sql
SELECT u.full_name, o.id AS order_id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
UNION
SELECT u.full_name, o.id AS order_id
FROM users u
RIGHT JOIN orders o ON o.user_id = u.id;
```

Kết quả gồm cả người dùng chưa có đơn lẫn đơn không có người dùng. Ở đây phải dùng `UNION` (không phải `UNION ALL`) để loại các dòng khớp xuất hiện ở cả hai phía.

### e) Chuyển cột thành dòng (unpivot)

Bảng có nhiều cột số điện thoại, muốn gom thành danh sách một cột.

```sql
SELECT id, phone_home AS phone FROM contacts
UNION ALL
SELECT id, phone_work AS phone FROM contacts
UNION ALL
SELECT id, phone_mobile AS phone FROM contacts;
```

## 7. UNION và JOIN

| Tiêu chí | UNION | JOIN |
|---|---|---|
| Cách nối | Nối thêm **dòng** (dọc) | Nối thêm **cột** (ngang) |
| Điều kiện nối | Không cần, chỉ cần cùng số cột và kiểu | Cần điều kiện `ON` |
| Dùng khi | Gộp dữ liệu **cùng dạng** từ nhiều nguồn | Ghép dữ liệu **liên quan** của nhiều bảng |

**Cách nhớ:** JOIN làm bảng **rộng ra**, UNION làm bảng **dài ra**.

## 8. Mở rộng: INTERSECT và EXCEPT

Từ **MySQL 8.0.31** có thêm 2 toán tử tập hợp cùng họ với UNION:

| Toán tử | Ý nghĩa | Ví dụ |
|---|---|---|
| `INTERSECT` | Chỉ lấy dòng có ở **cả hai** kết quả | Người vừa là khách vừa là nhà cung cấp |
| `EXCEPT` | Lấy dòng ở kết quả đầu mà **không có** ở kết quả sau | Khách hàng không phải là nhà cung cấp |

```sql
SELECT email FROM customers
EXCEPT
SELECT email FROM suppliers;
```

Với phiên bản MySQL cũ hơn, có thể thay bằng `JOIN`, `IN`, hoặc `NOT EXISTS`.

## Tổng kết

| Nội dung | Ghi chú |
|---|---|
| `UNION` | Gộp và **loại dòng trùng** |
| `UNION ALL` | Gộp và **giữ tất cả dòng**, nhanh hơn |
| Quy tắc | Cùng số cột, kiểu dữ liệu tương thích, ghép theo vị trí |
| `ORDER BY` / `LIMIT` | Đặt cuối cùng, áp dụng cho toàn bộ kết quả |
| Tên cột kết quả | Lấy theo `SELECT` đầu tiên |

**Các lỗi hay gặp:**
- Hai `SELECT` khác số cột (lỗi `different number of columns`).
- Ghép nhầm thứ tự cột nên dữ liệu bị lẫn lộn.
- Dùng `UNION` thay vì `UNION ALL` khi không cần loại trùng, làm truy vấn chậm không cần thiết.
- Đặt `ORDER BY` ở giữa các `SELECT` mà không bọc ngoặc.

**Cách nhớ nhanh:** UNION là "xếp chồng hai danh sách lên nhau". `UNION` thì bỏ dòng trùng, `UNION ALL` thì giữ hết.

Khi bạn sẵn sàng, mình có thể làm tiếp **Window function** cho giai đoạn 4.
