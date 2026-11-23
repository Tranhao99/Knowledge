# JOIN

## INNER JOIN

Chỉ lấy các dòng có dữ liệu khớp ở **cả hai bảng**. Dòng nào không khớp (ví dụ người dùng chưa có đơn hàng) sẽ bị loại bỏ.

```sql
SELECT u.full_name, o.id AS order_id, o.order_date
FROM users u
INNER JOIN orders o
    ON o.user_id = u.id;
```

**Use case phổ biến:**

- Hiển thị danh sách đơn hàng kèm tên người mua.
- Xem chi tiết đơn hàng gồm tên sản phẩm, số lượng, giá (nối `orders`, `order_items`, `products`).

```sql
SELECT o.id AS order_id, p.name, oi.quantity, oi.unit_price
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p ON p.id = oi.product_id
WHERE o.id = 1;
```

## LEFT JOIN

Lấy **tất cả dòng của bảng bên trái**, kèm dữ liệu khớp từ bảng bên phải. Không khớp thì các cột bên phải là `NULL`.

```sql
SELECT u.full_name, o.id AS order_id
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id;
```

**Use case phổ biến:**

- Tìm các dòng "chưa có dữ liệu liên quan": người dùng chưa từng mua hàng, sản phẩm chưa bán được.
- Thống kê đầy đủ, kể cả giá trị 0: số đơn hàng của mỗi người dùng, người chưa mua vẫn hiện số 0.

```sql
-- Người dùng chưa có đơn hàng nào
SELECT u.full_name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- Số đơn hàng của từng người dùng (kể cả 0)
SELECT u.full_name, COUNT(o.id) AS total_orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.full_name;
```

## RIGHT JOIN

Ngược lại với LEFT JOIN: lấy **tất cả dòng của bảng bên phải**, kèm dữ liệu khớp từ bảng bên trái. Không khớp thì cột bên trái là `NULL`.

```sql
SELECT u.full_name, o.id AS order_id
FROM users u
RIGHT JOIN orders o
    ON o.user_id = u.id;
```

**Use case phổ biến:**

- Kiểm tra **dữ liệu mồ côi**: đơn hàng không có người mua hợp lệ (thường xảy ra khi bảng chưa có khóa ngoại).
- Khi bảng cần giữ đầy đủ nằm bên phải câu lệnh có sẵn. Thực tế thường viết lại thành LEFT JOIN bằng cách đổi chỗ hai bảng cho dễ đọc.

```sql
-- Đơn hàng không có người mua tương ứng
SELECT o.id AS order_id
FROM users u
RIGHT JOIN orders o ON o.user_id = u.id
WHERE u.id IS NULL;
```

## Tổng kết

| JOIN | Giữ lại | Dùng khi |
|---|---|---|
| INNER JOIN | Chỉ dòng khớp ở cả hai bảng | Lấy dữ liệu liên quan đầy đủ |
| LEFT JOIN | Tất cả dòng bảng trái | Tìm dòng thiếu liên kết, thống kê kể cả 0 |
| RIGHT JOIN | Tất cả dòng bảng phải | Kiểm tra dữ liệu mồ côi, hiếm dùng |

**Mẹo:** Trong thực tế, hầu hết dùng `INNER JOIN` và `LEFT JOIN`. `RIGHT JOIN` có thể thay bằng `LEFT JOIN` bằng cách đổi vị trí hai bảng.
