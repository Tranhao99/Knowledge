# CASE WHEN

> **CASE WHEN** là câu lệnh điều kiện trong SQL, hoạt động giống `if - else if - else` trong lập trình. Nó kiểm tra từng điều kiện theo thứ tự và **trả về giá trị tương ứng** với điều kiện đúng đầu tiên. Dùng để phân loại, đổi nhãn hoặc tính toán theo điều kiện ngay trong câu truy vấn.

Các ví dụ dùng bảng `products` (`id`, `name`, `price`, `quantity`, `category_id`) và `orders` (`id`, `user_id`, `status`, `total_amount`, `order_date`).

## Cú pháp cơ bản

CASE WHEN có 2 dạng.

### 1. Searched CASE (dạng phổ biến, dùng với mọi điều kiện)

```sql
CASE
    WHEN điều_kiện_1 THEN kết_quả_1
    WHEN điều_kiện_2 THEN kết_quả_2
    ELSE kết_quả_mặc_định
END
```

**Ví dụ:** Phân loại sản phẩm theo mức giá.

```sql
SELECT
    name,
    price,
    CASE
        WHEN price < 100000 THEN 'Giá rẻ'
        WHEN price < 500000 THEN 'Tầm trung'
        ELSE 'Cao cấp'
    END AS price_level
FROM products;
```

### 2. Simple CASE (so sánh bằng một cột/biểu thức)

```sql
CASE cột
    WHEN giá_trị_1 THEN kết_quả_1
    WHEN giá_trị_2 THEN kết_quả_2
    ELSE kết_quả_mặc_định
END
```

**Ví dụ:** Đổi mã trạng thái đơn hàng sang chữ tiếng Việt.

```sql
SELECT
    id,
    CASE status
        WHEN 'PENDING' THEN 'Chờ xác nhận'
        WHEN 'PAID' THEN 'Đã thanh toán'
        WHEN 'SHIPPED' THEN 'Đang giao'
        WHEN 'CANCELLED' THEN 'Đã hủy'
        ELSE 'Không xác định'
    END AS status_text
FROM orders;
```

> **Lưu ý:** Simple CASE chỉ so sánh **bằng** (`=`). Muốn so sánh khoảng (`<`, `>`, `BETWEEN`) hoặc kết hợp `AND`/`OR` thì dùng Searched CASE.

## Hành vi cần nhớ

- **Dừng ở điều kiện đúng đầu tiên:** các `WHEN` phía sau không được xét nữa, nên **thứ tự điều kiện rất quan trọng**.
- **Không có `ELSE` thì trả về `NULL`** khi không điều kiện nào đúng.
- **Các kết quả nên cùng kiểu dữ liệu** (cùng là chuỗi hoặc cùng là số) để tránh lỗi hoặc kết quả bất ngờ.
- **Luôn kết thúc bằng `END`**, nên đặt tên cột kết quả bằng `AS`.

**Ví dụ sai thứ tự** (điều kiện rộng đặt trước làm điều kiện hẹp không bao giờ chạy tới):

```sql
-- Sai: giá 50.000 vẫn rơi vào 'Tầm trung' vì price < 500000 đúng trước
CASE
    WHEN price < 500000 THEN 'Tầm trung'
    WHEN price < 100000 THEN 'Giá rẻ'
END

-- Đúng: đi từ điều kiện hẹp đến rộng
CASE
    WHEN price < 100000 THEN 'Giá rẻ'
    WHEN price < 500000 THEN 'Tầm trung'
END
```

## Các vị trí dùng CASE WHEN

### 1. Trong SELECT: phân loại, gắn nhãn

**Cảnh báo tồn kho:**

```sql
SELECT
    name,
    quantity,
    CASE
        WHEN quantity = 0 THEN 'Hết hàng'
        WHEN quantity <= 10 THEN 'Sắp hết'
        ELSE 'Còn hàng'
    END AS stock_status
FROM products;
```

### 2. Trong ORDER BY: sắp xếp theo thứ tự tùy chỉnh

Đưa đơn chờ xác nhận lên đầu, rồi đến đã thanh toán, các đơn còn lại xuống cuối:

```sql
SELECT id, status, order_date
FROM orders
ORDER BY
    CASE status
        WHEN 'PENDING' THEN 1
        WHEN 'PAID' THEN 2
        ELSE 3
    END,
    order_date DESC;
```

### 3. Trong UPDATE: cập nhật nhiều giá trị khác nhau trong một lệnh

Giảm giá theo mức giá hiện tại:

```sql
UPDATE products
SET price = CASE
    WHEN price >= 500000 THEN price * 0.8
    WHEN price >= 200000 THEN price * 0.9
    ELSE price
END;
```

> **Cẩn thận:** Nhớ có `ELSE price` để các dòng không thuộc điều kiện nào **giữ nguyên giá trị**. Nếu thiếu, chúng sẽ bị gán thành `NULL`.

### 4. Kết hợp hàm tổng hợp (rất hay dùng cho báo cáo)

Cách này gọi là **conditional aggregation**: đếm hoặc cộng theo điều kiện, tạo nhiều cột thống kê chỉ trong một truy vấn.

**Thống kê số đơn theo từng trạng thái:**

```sql
SELECT
    COUNT(*) AS total_orders,
    SUM(CASE WHEN status = 'PAID' THEN 1 ELSE 0 END) AS paid_orders,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders
FROM orders;
```

**Doanh thu từng tháng, tách đơn thành công và đơn hủy:**

```sql
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(CASE WHEN status <> 'CANCELLED' THEN total_amount ELSE 0 END) AS revenue,
    SUM(CASE WHEN status = 'CANCELLED' THEN total_amount ELSE 0 END) AS lost_revenue
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m');
```

### 5. Trong WHERE: điều kiện động (ít dùng)

```sql
SELECT * FROM products
WHERE price > CASE
    WHEN category_id = 1 THEN 200000
    ELSE 100000
END;
```

## Use case phổ biến trong ứng dụng bán hàng

- **Gắn nhãn hiển thị:** trạng thái đơn hàng, mức tồn kho, phân hạng khách hàng (Vàng/Bạc/Đồng theo tổng chi tiêu).
- **Báo cáo thống kê:** đếm và cộng tiền theo trạng thái, theo loại, trong một truy vấn.
- **Sắp xếp theo thứ tự nghiệp vụ:** ưu tiên đơn cần xử lý lên đầu.
- **Cập nhật hàng loạt theo quy tắc:** áp giá, áp khuyến mãi theo nhiều mức.

**Ví dụ phân hạng khách hàng:**

```sql
SELECT
    u.full_name,
    SUM(o.total_amount) AS total_spent,
    CASE
        WHEN SUM(o.total_amount) >= 10000000 THEN 'Vàng'
        WHEN SUM(o.total_amount) >= 5000000 THEN 'Bạc'
        ELSE 'Đồng'
    END AS member_rank
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status <> 'CANCELLED'
GROUP BY u.id, u.full_name;
```

## Tổng kết

| Đặc điểm | Ghi chú |
|---|---|
| 2 dạng | Searched CASE (điều kiện bất kỳ), Simple CASE (so sánh bằng) |
| Thứ tự | Dừng ở điều kiện đúng đầu tiên, đi từ hẹp đến rộng |
| Không có ELSE | Trả về `NULL` |
| Dùng ở đâu | `SELECT`, `ORDER BY`, `UPDATE`, `WHERE`, kết hợp `SUM`/`COUNT` |

**Cách nhớ nhanh:** CASE WHEN là "if - else if - else" của SQL. Hãy viết điều kiện hẹp trước, nhớ `ELSE` và nhớ `END`.

Khi bạn sẵn sàng, mình có thể làm tiếp các nội dung khác của giai đoạn 4 như `GROUP BY`/`HAVING`, Subquery hay Window function.
