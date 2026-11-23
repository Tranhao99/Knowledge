# GROUP BY, HAVING và các hàm tổng hợp

> **Hàm tổng hợp (aggregate function)** gom nhiều dòng dữ liệu thành **một giá trị** (đếm, cộng, trung bình...). **GROUP BY** chia dữ liệu thành các nhóm để tính toán riêng cho từng nhóm. **HAVING** lọc lại các nhóm sau khi đã tính toán. Ba thứ này thường đi cùng nhau để làm báo cáo, thống kê.

Các ví dụ dùng bảng `products` (`id`, `name`, `price`, `quantity`, `category_id`), `orders` (`id`, `user_id`, `status`, `total_amount`, `order_date`) và `users` (`id`, `full_name`).

## 1. Các hàm tổng hợp

| Hàm | Ý nghĩa |
|---|---|
| `COUNT()` | Đếm số dòng |
| `SUM()` | Tính tổng |
| `AVG()` | Tính trung bình |
| `MAX()` | Lấy giá trị lớn nhất |
| `MIN()` | Lấy giá trị nhỏ nhất |

**Ví dụ:** Thống kê toàn bộ bảng `products`.

```sql
SELECT
    COUNT(*) AS total_products,
    SUM(quantity) AS total_stock,
    AVG(price) AS avg_price,
    MAX(price) AS max_price,
    MIN(price) AS min_price
FROM products;
```

Kết quả chỉ có **1 dòng**, vì tất cả dòng đã được gom lại thành một.

### Phân biệt các cách dùng COUNT

```sql
SELECT
    COUNT(*) AS all_rows,
    COUNT(category_id) AS has_category,
    COUNT(DISTINCT category_id) AS unique_categories
FROM products;
```

- `COUNT(*)`: đếm **tất cả dòng**, kể cả dòng có giá trị `NULL`.
- `COUNT(cột)`: chỉ đếm dòng mà cột đó **khác `NULL`**.
- `COUNT(DISTINCT cột)`: đếm số giá trị **không trùng nhau**.

### Những điều cần nhớ về hàm tổng hợp

- **Bỏ qua `NULL`:** `SUM`, `AVG`, `MAX`, `MIN`, `COUNT(cột)` đều không tính các giá trị `NULL`. Riêng `COUNT(*)` thì tính.
- **Bảng không có dòng nào:** `COUNT` trả về `0`, còn `SUM`, `AVG`, `MAX`, `MIN` trả về `NULL`. Dùng `COALESCE` để đổi `NULL` thành `0`.
- **Không dùng trong `WHERE`:** vì `WHERE` chạy trước khi gom nhóm (xem mục HAVING).

```sql
SELECT COALESCE(SUM(total_amount), 0) AS revenue
FROM orders
WHERE status = 'PAID';
```

## 2. GROUP BY

Chia các dòng thành **từng nhóm có cùng giá trị** ở cột chỉ định, rồi áp dụng hàm tổng hợp cho **mỗi nhóm**.

**Ví dụ:** Đếm số sản phẩm theo từng danh mục.

```sql
SELECT category_id, COUNT(*) AS total_products
FROM products
GROUP BY category_id;
```

Cách hoạt động:

| category_id | name |  | category_id | total_products |
|---|---|---|---|---|
| 1 | Áo thun |  | 1 | 2 |
| 1 | Áo khoác | → | 2 | 1 |
| 2 | Quần jean |  |  |  |

### Nhóm theo nhiều cột

Mỗi **tổ hợp giá trị** của các cột là một nhóm.

```sql
SELECT category_id, status, COUNT(*) AS total
FROM orders
GROUP BY category_id, status;
```

### Kết hợp với JOIN

**Ví dụ:** Số đơn hàng và tổng tiền của từng người dùng.

```sql
SELECT
    u.full_name,
    COUNT(o.id) AS total_orders,
    SUM(o.total_amount) AS total_spent
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.full_name;
```

### Quy tắc quan trọng

> Mọi cột trong `SELECT` phải là **cột nằm trong `GROUP BY`** hoặc là **kết quả của hàm tổng hợp**.

```sql
-- Sai: name không nằm trong GROUP BY và không có hàm tổng hợp
SELECT category_id, name, COUNT(*)
FROM products
GROUP BY category_id;

-- Đúng
SELECT category_id, COUNT(*)
FROM products
GROUP BY category_id;
```

MySQL từ bản 5.7 mặc định bật chế độ `ONLY_FULL_GROUP_BY` nên câu sai ở trên sẽ báo lỗi.

### Nhóm theo biểu thức

Có thể nhóm theo kết quả của hàm, ví dụ doanh thu từng tháng:

```sql
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

## 3. HAVING

**Lọc các nhóm** sau khi đã `GROUP BY` và tính xong hàm tổng hợp. Dùng khi điều kiện lọc cần dựa vào **kết quả tổng hợp**.

**Ví dụ:** Chỉ lấy danh mục có từ 5 sản phẩm trở lên.

```sql
SELECT category_id, COUNT(*) AS total_products
FROM products
GROUP BY category_id
HAVING COUNT(*) >= 5;
```

**Ví dụ:** Khách hàng đã chi tiêu trên 5 triệu.

```sql
SELECT u.full_name, SUM(o.total_amount) AS total_spent
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.full_name
HAVING SUM(o.total_amount) > 5000000
ORDER BY total_spent DESC;
```

### So sánh WHERE và HAVING

| Tiêu chí | WHERE | HAVING |
|---|---|---|
| Lọc cái gì | Từng **dòng** | Từng **nhóm** |
| Chạy khi nào | **Trước** `GROUP BY` | **Sau** `GROUP BY` |
| Dùng hàm tổng hợp | Không | Có |
| Hiệu năng | Nhanh hơn (giảm dòng sớm) | Chậm hơn |

**Dùng cả hai cùng lúc:** `WHERE` lọc dòng trước, `HAVING` lọc nhóm sau.

```sql
-- Bỏ đơn đã hủy (WHERE), rồi chỉ lấy khách có từ 3 đơn trở lên (HAVING)
SELECT user_id, COUNT(*) AS total_orders
FROM orders
WHERE status <> 'CANCELLED'
GROUP BY user_id
HAVING COUNT(*) >= 3;
```

> **Mẹo hiệu năng:** Điều kiện nào lọc được bằng `WHERE` thì nên đặt ở `WHERE`, đừng đưa vào `HAVING`.

## 4. Thứ tự thực thi của câu lệnh

Thứ tự **viết** khác với thứ tự MySQL **chạy**:

1. `FROM` / `JOIN`: lấy và nối bảng
2. `WHERE`: lọc dòng
3. `GROUP BY`: gom nhóm
4. `HAVING`: lọc nhóm
5. `SELECT`: tính cột kết quả
6. `ORDER BY`: sắp xếp
7. `LIMIT`: giới hạn số dòng

Nhờ đó dễ hiểu vì sao `WHERE` không dùng được hàm tổng hợp (lúc đó chưa gom nhóm), còn `HAVING` thì dùng được.

## 5. Use case phổ biến trong ứng dụng bán hàng

**Top 5 sản phẩm bán chạy:**

```sql
SELECT p.name, SUM(oi.quantity) AS total_sold
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.id, p.name
ORDER BY total_sold DESC
LIMIT 5;
```

**Doanh thu theo tháng, chỉ tính đơn đã thanh toán:**

```sql
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS revenue
FROM orders
WHERE status = 'PAID'
GROUP BY DATE_FORMAT(order_date, '%Y-%m');
```

**Tìm dữ liệu trùng (ví dụ email bị trùng):**

```sql
SELECT email, COUNT(*) AS duplicates
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Danh mục có giá trung bình trên 200.000:**

```sql
SELECT category_id, AVG(price) AS avg_price
FROM products
GROUP BY category_id
HAVING AVG(price) > 200000;
```

**Đếm cả người dùng chưa có đơn hàng (LEFT JOIN):**

```sql
SELECT u.full_name, COUNT(o.id) AS total_orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.full_name;
```

Ở ví dụ cuối, phải dùng `COUNT(o.id)` thay vì `COUNT(*)`. Nếu dùng `COUNT(*)`, người dùng chưa có đơn vẫn bị đếm là 1 (vì có 1 dòng với các cột bên đơn hàng là `NULL`), còn `COUNT(o.id)` cho ra đúng 0.

## Tổng kết

| Thành phần | Vai trò | Lưu ý |
|---|---|---|
| Hàm tổng hợp | Gom nhiều dòng thành 1 giá trị | Bỏ qua `NULL`, riêng `COUNT(*)` thì không |
| `GROUP BY` | Chia dữ liệu thành các nhóm | Cột trong `SELECT` phải nằm trong `GROUP BY` hoặc là hàm tổng hợp |
| `HAVING` | Lọc nhóm sau khi tổng hợp | Dùng được hàm tổng hợp, chậm hơn `WHERE` |

**Các lỗi hay gặp:**

- Dùng hàm tổng hợp trong `WHERE` (phải chuyển sang `HAVING`).
- `SELECT` cột không có trong `GROUP BY`.
- Dùng `COUNT(*)` với `LEFT JOIN` làm kết quả bị đếm dư.
- Quên `COALESCE` khi `SUM` có thể trả về `NULL`.

**Cách nhớ nhanh:** `WHERE` lọc **dòng** trước khi gom, `GROUP BY` **gom** thành nhóm, `HAVING` lọc **nhóm** sau khi gom.

Khi bạn sẵn sàng, mình có thể làm tiếp **Subquery** hoặc **Window function** cho giai đoạn 4.
