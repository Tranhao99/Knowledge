# Subquery

> **Subquery (truy vấn con)** là một câu `SELECT` được đặt **bên trong** một câu truy vấn khác (gọi là truy vấn cha). Truy vấn con chạy để cho ra kết quả, rồi truy vấn cha dùng kết quả đó. Subquery luôn nằm trong cặp ngoặc đơn `( )`.

Các ví dụ dùng bảng `products` (`id`, `name`, `price`, `quantity`, `category_id`), `users` (`id`, `full_name`), `orders` (`id`, `user_id`, `status`, `total_amount`, `order_date`) và `order_items` (`order_id`, `product_id`, `quantity`).

## 1. Subquery trong WHERE

Dùng kết quả của truy vấn con làm **điều kiện lọc**. Đây là cách dùng phổ biến nhất.

### a) Subquery trả về 1 giá trị (scalar subquery)

Dùng với các toán tử so sánh `=`, `>`, `<`, `>=`, `<=`, `<>`.

**Ví dụ:** Sản phẩm có giá cao hơn giá trung bình.

```sql
SELECT name, price
FROM products
WHERE price > (
    SELECT AVG(price)
    FROM products
);
```

Truy vấn con tính ra một con số (giá trung bình), truy vấn cha dùng con số đó để lọc.

> **Lưu ý:** Nếu truy vấn con trả về **nhiều hơn 1 dòng** mà dùng toán tử so sánh thường, MySQL sẽ báo lỗi `Subquery returns more than 1 row`.

### b) Subquery trả về nhiều giá trị (dùng IN / NOT IN)

**Ví dụ:** Người dùng đã từng đặt hàng.

```sql
SELECT full_name
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
```

**Ví dụ:** Sản phẩm chưa từng được bán.

```sql
SELECT name
FROM products
WHERE id NOT IN (
    SELECT product_id
    FROM order_items
    WHERE product_id IS NOT NULL
);
```

> **Cẩn thận với `NOT IN`:** Nếu danh sách truy vấn con có **bất kỳ giá trị `NULL` nào**, `NOT IN` sẽ không trả về dòng nào cả. Vì vậy phải thêm `WHERE cột IS NOT NULL` như trên, hoặc dùng `NOT EXISTS` (an toàn hơn).

### c) EXISTS / NOT EXISTS

Chỉ kiểm tra **có tồn tại dòng nào hay không**, không quan tâm giá trị cụ thể.

**Ví dụ:** Người dùng có ít nhất 1 đơn hàng.

```sql
SELECT u.full_name
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
);
```

**Ví dụ:** Người dùng chưa có đơn hàng nào.

```sql
SELECT u.full_name
FROM users u
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
);
```

`SELECT 1` chỉ là quy ước, vì `EXISTS` chỉ quan tâm có dòng hay không, nên chọn cột nào cũng được.

### d) ANY / ALL

So sánh với **một trong** (`ANY`) hoặc **tất cả** (`ALL`) giá trị mà truy vấn con trả về.

```sql
-- Giá cao hơn TẤT CẢ sản phẩm của danh mục 2
SELECT name, price
FROM products
WHERE price > ALL (
    SELECT price FROM products WHERE category_id = 2
);

-- Giá cao hơn ÍT NHẤT MỘT sản phẩm của danh mục 2
SELECT name, price
FROM products
WHERE price > ANY (
    SELECT price FROM products WHERE category_id = 2
);
```

## 2. Subquery trong FROM (Derived table)

Kết quả của truy vấn con được dùng như **một bảng tạm**. Bắt buộc phải đặt **bí danh (alias)** cho bảng tạm này.

**Ví dụ:** Tổng chi tiêu của từng người dùng, rồi chỉ lấy người chi trên mức trung bình.

```sql
SELECT t.user_id, t.total_spent
FROM (
    SELECT user_id, SUM(total_amount) AS total_spent
    FROM orders
    GROUP BY user_id
) AS t
WHERE t.total_spent > 5000000;
```

**Ví dụ:** Tính trung bình của các giá trị đã tổng hợp (không thể gọi `AVG(SUM(...))` trực tiếp).

```sql
SELECT AVG(t.total_spent) AS avg_spent_per_user
FROM (
    SELECT user_id, SUM(total_amount) AS total_spent
    FROM orders
    GROUP BY user_id
) AS t;
```

## 3. Subquery trong SELECT

Truy vấn con đóng vai trò **một cột** trong kết quả. Bắt buộc chỉ trả về **đúng 1 giá trị** cho mỗi dòng.

**Ví dụ:** Mỗi người dùng kèm số đơn hàng.

```sql
SELECT
    u.full_name,
    (SELECT COUNT(*)
     FROM orders o
     WHERE o.user_id = u.id) AS total_orders
FROM users u;
```

**Ví dụ:** Mỗi sản phẩm kèm chênh lệch so với giá trung bình.

```sql
SELECT
    name,
    price,
    price - (SELECT AVG(price) FROM products) AS diff_from_avg
FROM products;
```

## 4. Subquery tương quan (Correlated subquery)

Truy vấn con **tham chiếu đến cột của truy vấn cha**, nên phải chạy lại cho **từng dòng** của truy vấn cha. Các ví dụ dùng `EXISTS` và `SELECT` ở trên đều là dạng này (xuất hiện `u.id`, `o.user_id = u.id`).

**Ví dụ:** Sản phẩm có giá cao hơn giá trung bình **của chính danh mục đó**.

```sql
SELECT p.name, p.price, p.category_id
FROM products p
WHERE p.price > (
    SELECT AVG(p2.price)
    FROM products p2
    WHERE p2.category_id = p.category_id
);
```

| Loại | Đặc điểm |
|---|---|
| Subquery thường | Độc lập, chạy **một lần** rồi truyền kết quả cho truy vấn cha |
| Subquery tương quan | Phụ thuộc truy vấn cha, chạy **lặp lại theo từng dòng**, có thể chậm với dữ liệu lớn |

## 5. Subquery với INSERT, UPDATE, DELETE

**INSERT ... SELECT:** Sao chép dữ liệu từ bảng này sang bảng khác.

```sql
INSERT INTO vip_customers (user_id, total_spent)
SELECT user_id, SUM(total_amount)
FROM orders
GROUP BY user_id
HAVING SUM(total_amount) >= 10000000;
```

**UPDATE:** Đặt cờ cho sản phẩm đã từng bán.

```sql
UPDATE products
SET is_sold = TRUE
WHERE id IN (
    SELECT product_id FROM order_items
);
```

**DELETE:** Xóa người dùng chưa có đơn hàng.

```sql
DELETE FROM users
WHERE id NOT IN (
    SELECT user_id FROM orders WHERE user_id IS NOT NULL
);
```

> **Giới hạn của MySQL:** Không được dùng subquery **đọc từ chính bảng đang `UPDATE`/`DELETE`** (lỗi 1093). Cách khắc phục là bọc truy vấn con thêm một lớp bảng tạm:

```sql
DELETE FROM products
WHERE price < (
    SELECT avg_price FROM (
        SELECT AVG(price) AS avg_price FROM products
    ) AS t
);
```

## 6. Subquery và JOIN: dùng cái nào?

Nhiều bài toán làm được bằng cả hai cách.

**Cách 1: Subquery**

```sql
SELECT full_name
FROM users
WHERE id IN (SELECT user_id FROM orders);
```

**Cách 2: JOIN**

```sql
SELECT DISTINCT u.full_name
FROM users u
JOIN orders o ON o.user_id = u.id;
```

| Tiêu chí | Subquery | JOIN |
|---|---|---|
| Dễ đọc | Dễ hiểu logic "lọc theo kết quả của truy vấn khác" | Dễ đọc khi cần lấy cột từ nhiều bảng |
| Lấy cột từ bảng con | Không (chỉ để lọc hoặc tính) | Có |
| Hiệu năng | Thường tương đương trên MySQL 8, subquery tương quan có thể chậm | Thường ổn định và dễ tối ưu bằng index |
| Phù hợp | So sánh với giá trị tổng hợp, kiểm tra tồn tại (`EXISTS`) | Cần hiển thị dữ liệu từ nhiều bảng |

**Quy tắc chọn nhanh:**
- Cần **hiển thị cột** của bảng liên quan: dùng `JOIN`.
- Chỉ cần **lọc** theo sự tồn tại hoặc so sánh với giá trị tổng hợp: dùng subquery (`EXISTS`, `IN`, `> (SELECT AVG...)`).
- Subquery tương quan chạy chậm: thử viết lại bằng `JOIN` hoặc `GROUP BY`.

## 7. Use case phổ biến trong ứng dụng bán hàng

- **So sánh với mức trung bình hoặc cao nhất:** sản phẩm đắt nhất, đơn hàng có giá trị trên trung bình.
- **Tìm dữ liệu "không có liên kết":** khách chưa mua hàng, sản phẩm chưa bán được (`NOT EXISTS`).
- **Tổng hợp nhiều tầng:** tính trung bình của tổng chi tiêu từng khách (subquery trong `FROM`).
- **Thêm cột thống kê vào danh sách:** mỗi khách kèm số đơn, mỗi sản phẩm kèm số lượng đã bán (subquery trong `SELECT`).

**Ví dụ:** Sản phẩm có giá cao nhất.

```sql
SELECT name, price
FROM products
WHERE price = (SELECT MAX(price) FROM products);
```

## Tổng kết

| Vị trí | Trả về | Dùng để |
|---|---|---|
| `WHERE` | 1 giá trị hoặc 1 danh sách | Lọc dòng |
| `FROM` | Một bảng tạm (cần alias) | Tổng hợp nhiều tầng |
| `SELECT` | Đúng 1 giá trị mỗi dòng | Thêm cột tính toán |
| `EXISTS` | Có / không có dòng | Kiểm tra tồn tại |

**Các lỗi hay gặp:**
- Dùng `=` với subquery trả về nhiều dòng (phải dùng `IN`).
- `NOT IN` gặp giá trị `NULL` nên không ra kết quả (dùng `NOT EXISTS`).
- Quên alias cho subquery trong `FROM`.
- Subquery đọc chính bảng đang `UPDATE`/`DELETE` trong MySQL.

**Cách nhớ nhanh:** Subquery là "câu hỏi nhỏ trả lời trước, câu hỏi lớn dùng đáp án đó". Nó có thể thay thế cho một giá trị, một danh sách hoặc một bảng tạm.

**Một điều cần sửa:** Trong nội dung GROUP BY lúc trước, ví dụ "Nhóm theo nhiều cột" tôi viết `GROUP BY category_id, status` trên bảng `orders` là **sai**, vì bảng `orders` không có cột `category_id`. Bạn nên sửa thành `GROUP BY user_id, status` (đếm số đơn theo từng người dùng và từng trạng thái):

```sql
SELECT user_id, status, COUNT(*) AS total
FROM orders
GROUP BY user_id, status;
```

Khi bạn sẵn sàng, mình có thể làm tiếp **Window function** cho giai đoạn 4.
