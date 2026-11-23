# Window function

> **Window function (hàm cửa sổ)** là hàm tính toán trên một **tập các dòng liên quan** đến dòng hiện tại, nhưng **vẫn giữ nguyên từng dòng** trong kết quả. Nhờ đó vừa xem được chi tiết, vừa xem được số liệu tổng hợp (xếp hạng, lũy kế, so sánh dòng trước) trong cùng một truy vấn.

**Yêu cầu:** MySQL **8.0 trở lên**.

Các ví dụ dùng bảng `products` (`id`, `name`, `price`, `category_id`) và `orders` (`id`, `user_id`, `status`, `total_amount`, `order_date`).

## 1. Khác biệt với GROUP BY

| Tiêu chí | GROUP BY | Window function |
|---|---|---|
| Số dòng kết quả | **Gộp** mỗi nhóm còn 1 dòng | **Giữ nguyên** số dòng |
| Xem chi tiết từng dòng | Không | Có |

```sql
-- GROUP BY: mất chi tiết sản phẩm
SELECT category_id, AVG(price) AS avg_price
FROM products
GROUP BY category_id;

-- Window function: giữ từng sản phẩm, kèm giá trung bình danh mục
SELECT
    name,
    price,
    AVG(price) OVER (PARTITION BY category_id) AS avg_category_price
FROM products;
```

## 2. Cú pháp tổng quát

```sql
tên_hàm(...) OVER (
    PARTITION BY cột_chia_nhóm
    ORDER BY cột_sắp_xếp
)
```

- **`OVER (...)`**: báo hiệu đây là window function.
- **`PARTITION BY`**: chia dòng thành các nhóm (giống `GROUP BY` nhưng không gộp dòng). Bỏ đi thì cả bảng là một nhóm.
- **`ORDER BY`**: sắp xếp dòng bên trong mỗi nhóm.

## 3. Hàm xếp hạng: ROW_NUMBER, RANK, DENSE_RANK

Giả sử các sản phẩm cùng danh mục có giá: 500, 400, 400, 300.

| price | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|
| 500 | 1 | 1 | 1 |
| 400 | 2 | 2 | 2 |
| 400 | 3 | 2 | 2 |
| 300 | 4 | **4** | **3** |

- **`ROW_NUMBER()`**: đánh số liên tục, không bao giờ trùng.
- **`RANK()`**: giá trị bằng nhau thì cùng hạng, hạng kế tiếp **nhảy cách**.
- **`DENSE_RANK()`**: giá trị bằng nhau thì cùng hạng, hạng kế tiếp **không nhảy cách**.

```sql
SELECT
    name,
    category_id,
    price,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS row_num,
    RANK()       OVER (PARTITION BY category_id ORDER BY price DESC) AS rank_no,
    DENSE_RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS dense_rank_no
FROM products;
```

### Use case: Top N trong mỗi nhóm

Bài toán rất hay gặp: lấy **3 sản phẩm đắt nhất của từng danh mục**. Window function không dùng trực tiếp trong `WHERE` được, nên phải bọc bằng CTE.

```sql
WITH ranked AS (
    SELECT
        name,
        category_id,
        price,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS rn
    FROM products
)
SELECT name, category_id, price
FROM ranked
WHERE rn <= 3;
```

> **Chọn hàm nào?** Dùng `ROW_NUMBER` khi cần **đúng N dòng**. Dùng `RANK` hoặc `DENSE_RANK` khi muốn các dòng bằng điểm được xếp ngang nhau.

### Use case: Bản ghi mới nhất của mỗi nhóm

```sql
WITH latest AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) AS rn
    FROM orders
)
SELECT *
FROM latest
WHERE rn = 1;
```

Kết quả là đơn hàng gần nhất của mỗi khách.

## 4. Hàm tổng hợp kết hợp OVER

Các hàm quen thuộc `SUM`, `AVG`, `COUNT`, `MAX`, `MIN` khi thêm `OVER (...)` sẽ trở thành window function.

### Tổng theo nhóm kèm chi tiết

```sql
SELECT
    user_id,
    id AS order_id,
    total_amount,
    SUM(total_amount) OVER (PARTITION BY user_id) AS user_total,
    COUNT(*) OVER (PARTITION BY user_id) AS user_order_count
FROM orders;
```

### Tỷ lệ phần trăm trên tổng

```sql
SELECT
    id,
    total_amount,
    ROUND(total_amount / SUM(total_amount) OVER () * 100, 2) AS percent_of_total
FROM orders;
```

`OVER ()` để trống nghĩa là cửa sổ là toàn bộ bảng.

### Tổng lũy kế (running total)

```sql
SELECT
    order_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY order_date, id) AS running_total
FROM orders;
```

Mỗi dòng hiển thị tổng doanh thu **từ đầu đến dòng đó**.

> **Lưu ý:** Nên thêm `id` vào `ORDER BY` như trên. Nếu nhiều dòng có cùng `order_date`, MySQL sẽ tính chung chúng nên cho ra cùng một giá trị lũy kế.

## 5. LAG và LEAD: so sánh với dòng trước/sau

- **`LAG(cột)`**: lấy giá trị của dòng **trước**.
- **`LEAD(cột)`**: lấy giá trị của dòng **sau**.
- Dòng không tồn tại thì trả về `NULL`.

**Ví dụ:** So sánh doanh thu từng tháng với tháng trước.

```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        SUM(total_amount) AS revenue
    FROM orders
    GROUP BY DATE_FORMAT(order_date, '%Y-%m')
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS diff
FROM monthly;
```

**Ví dụ:** Số ngày giữa 2 lần đặt hàng liên tiếp của mỗi khách.

```sql
SELECT
    user_id,
    order_date,
    DATEDIFF(
        order_date,
        LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date)
    ) AS days_since_last_order
FROM orders;
```

## 6. Hạn chế cần nhớ

- **Không dùng window function trong `WHERE`, `GROUP BY`, `HAVING`.** Muốn lọc theo kết quả của nó, hãy bọc bằng **CTE hoặc subquery** (như ví dụ Top N).
- **Chỉ xuất hiện trong `SELECT` và `ORDER BY`.**
- **Thứ tự thực thi:** `FROM/JOIN` → `WHERE` → `GROUP BY` → `HAVING` → **window function** → `SELECT` → `ORDER BY` → `LIMIT`.
- **Hiệu năng:** Window function phải sắp xếp dữ liệu, nên với bảng lớn cần có index phù hợp với cột trong `PARTITION BY` và `ORDER BY`.

## Tổng kết

| Nhóm | Hàm | Dùng để |
|---|---|---|
| Xếp hạng | `ROW_NUMBER`, `RANK`, `DENSE_RANK` | Đánh số, xếp hạng, Top N theo nhóm |
| Tổng hợp | `SUM`, `AVG`, `COUNT` + `OVER` | Tổng theo nhóm, lũy kế, tỷ lệ % |
| Giá trị | `LAG`, `LEAD` | So sánh với dòng trước/sau |

**Các lỗi hay gặp:**
- Dùng window function trong `WHERE` (phải bọc CTE hoặc subquery).
- Nhầm `RANK` với `DENSE_RANK`, hoặc dùng `ROW_NUMBER` khi muốn các dòng bằng điểm cùng hạng.
- Tổng lũy kế bị trùng giá trị do `ORDER BY` có nhiều dòng bằng nhau.
- Chạy trên MySQL 5.7 (không hỗ trợ).

**Cách nhớ nhanh:** `GROUP BY` là "gộp lại rồi tính", còn window function là "tính nhưng không gộp". `PARTITION BY` chia nhóm, `ORDER BY` xếp thứ tự trong nhóm.

Bản này đã lược bỏ `NTILE`, khung cửa sổ (`ROWS BETWEEN`), `FIRST_VALUE`/`LAST_VALUE` và `WINDOW` clause vì ít dùng hơn. Khi cần, mình có thể bổ sung riêng từng phần.
