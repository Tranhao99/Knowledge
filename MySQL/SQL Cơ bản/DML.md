# DML

> **DML (Data Manipulation Language - Ngôn ngữ thao tác dữ liệu)** là nhóm lệnh SQL dùng để **thêm, xem, sửa, xóa dữ liệu** bên trong các bảng. Nếu DDL là xây "căn phòng", thì DML là việc cất đồ vào phòng, xem đồ, đổi đồ và bỏ đồ đi.

Các ví dụ bên dưới dùng bảng `products` (sản phẩm) gồm các cột: `id`, `name`, `price`, `quantity`.

## 1. INSERT

Dùng để **thêm dòng dữ liệu mới** vào bảng.

**Thêm 1 dòng:**

```sql
INSERT INTO products (name, price, quantity)
VALUES ('Áo thun', 150000, 20);
```

**Thêm nhiều dòng cùng lúc:**

```sql
INSERT INTO products (name, price, quantity)
VALUES
    ('Quần jean', 350000, 15),
    ('Giày thể thao', 800000, 10),
    ('Mũ lưỡi trai', 90000, 30);
```

> **Lưu ý:** Cột `id` đã có `AUTO_INCREMENT` nên không cần ghi, MySQL tự tăng. Thứ tự giá trị trong `VALUES` phải khớp với thứ tự cột đã khai báo.

## 2. SELECT

Dùng để **truy vấn (xem) dữ liệu**. Đây là lệnh dùng nhiều nhất.

**Lấy tất cả cột, tất cả dòng:**

```sql
SELECT * FROM products;
```

**Chỉ lấy một số cột:**

```sql
SELECT name, price
FROM products;
```

**Lọc dữ liệu với WHERE:**

```sql
SELECT * FROM products
WHERE price > 200000;
```

Một số cách lọc hay gặp:

```sql
-- Kết hợp nhiều điều kiện (AND, OR)
SELECT * FROM products
WHERE price > 100000 AND quantity > 10;

-- Tìm theo từ khóa (LIKE, % là ký tự đại diện)
SELECT * FROM products
WHERE name LIKE '%thun%';

-- Nằm trong khoảng (BETWEEN)
SELECT * FROM products
WHERE price BETWEEN 100000 AND 500000;

-- Nằm trong danh sách (IN)
SELECT * FROM products
WHERE id IN (1, 3, 5);
```

**Sắp xếp với ORDER BY** (`ASC` tăng dần, `DESC` giảm dần):

```sql
SELECT * FROM products
ORDER BY price DESC;
```

**Giới hạn số dòng với LIMIT:**

```sql
SELECT * FROM products
ORDER BY price DESC
LIMIT 5;
```

**Thống kê nhanh với hàm tổng hợp** (`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`):

```sql
SELECT COUNT(*) AS total_products
FROM products;
```

> **Thứ tự viết lệnh:** `SELECT` → `FROM` → `WHERE` → `ORDER BY` → `LIMIT`.

## 3. UPDATE

Dùng để **sửa dữ liệu** của các dòng đã có.

**Sửa 1 dòng:**

```sql
UPDATE products
SET price = 160000
WHERE id = 1;
```

**Sửa nhiều cột cùng lúc:**

```sql
UPDATE products
SET price = 170000, quantity = 25
WHERE id = 1;
```

**Sửa nhiều dòng theo điều kiện** (ví dụ giảm giá 10% cho sản phẩm trên 500.000):

```sql
UPDATE products
SET price = price * 0.9
WHERE price > 500000;
```

> **Cảnh báo:** Nếu quên `WHERE`, **toàn bộ dòng trong bảng sẽ bị sửa**. Luôn kiểm tra điều kiện bằng `SELECT` trước khi chạy `UPDATE`.

## 4. DELETE

Dùng để **xóa các dòng dữ liệu** khỏi bảng (bảng vẫn còn).

**Xóa theo điều kiện:**

```sql
DELETE FROM products
WHERE id = 5;
```

**Xóa nhiều dòng:**

```sql
DELETE FROM products
WHERE quantity = 0;
```

> **Cảnh báo:** Giống `UPDATE`, nếu quên `WHERE` thì **xóa sạch toàn bộ dữ liệu** của bảng. Mẹo an toàn: viết `SELECT * FROM ... WHERE ...` trước, kiểm tra kết quả đúng rồi mới đổi thành `DELETE`.

## Tổng kết

| Lệnh | Làm gì | Ví dụ ngắn |
|---|---|---|
| INSERT | Thêm dữ liệu mới | `INSERT INTO products (...) VALUES (...)` |
| SELECT | Xem dữ liệu | `SELECT * FROM products` |
| UPDATE | Sửa dữ liệu | `UPDATE products SET ... WHERE ...` |
| DELETE | Xóa dữ liệu | `DELETE FROM products WHERE ...` |

**Cách nhớ nhanh:** 4 lệnh này tương ứng với 4 thao tác **CRUD** mà ứng dụng Spring Boot nào cũng cần: **C**reate (INSERT), **R**ead (SELECT), **U**pdate (UPDATE), **D**elete (DELETE).

**Khác biệt với DDL:** DML thao tác trên **dữ liệu**, và có thể hoàn tác (rollback) nếu dùng trong transaction. DDL thao tác trên **cấu trúc**, và không thể hoàn tác.
