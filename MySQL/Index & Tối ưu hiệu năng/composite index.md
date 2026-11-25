# Composite Index

> **Composite index (index tổng hợp)** là index được tạo trên **2 cột trở lên** trong cùng một bảng. Nó phù hợp khi các truy vấn thường lọc hoặc sắp xếp theo nhiều cột cùng lúc.

Các ví dụ dùng bảng `orders` (`id`, `user_id`, `status`, `order_date`, `total_amount`) và index:

```sql
CREATE INDEX idx_orders_user_status_date
ON orders(user_id, status, order_date);
```

## Cấu trúc B-Tree của Composite Index

- **Khác biệt với index 1 cột:** mỗi khóa trong cây là **một bộ giá trị** (user_id, status, order_date) chứ không phải một giá trị đơn.
- **Cách sắp xếp:** sắp theo cột thứ nhất trước. Những dòng cùng cột thứ nhất mới được sắp tiếp theo cột thứ hai, rồi mới đến cột thứ ba. Giống như danh bạ sắp theo họ, cùng họ thì sắp theo tên, cùng tên thì sắp theo ngày sinh.
- **Hệ quả:** dữ liệu chỉ "có thứ tự" theo đúng thứ tự cột khai báo. Cột sau chỉ có thứ tự **trong phạm vi cùng giá trị của các cột phía trước**.

```
(user_id, status,    order_date)
(1,       'PAID',    2026-01-05)
(1,       'PAID',    2026-02-10)
(1,       'PENDING', 2026-01-20)   ← cùng user_id=1, sắp tiếp theo status
(2,       'PAID',    2026-01-03)
(2,       'SHIPPED', 2026-03-01)
```

Tìm `user_id = 1 AND status = 'PAID'`: nhảy thẳng đến vùng user_id = 1, rồi đến vùng 'PAID' trong đó. Tìm riêng `status = 'PAID'`: các dòng 'PAID' nằm **rải rác** ở nhiều vùng user_id, nên không tận dụng được thứ tự.

## Quy tắc quan trọng nhất: Leftmost Prefix

MySQL chỉ dùng được composite index khi truy vấn dùng **các cột liên tiếp tính từ cột ngoài cùng bên trái**. Bỏ qua một cột giữa chừng thì các cột phía sau không còn dùng được để tìm kiếm.

| Điều kiện `WHERE` | Dùng được index? |
|---|---|
| `user_id` | Có (1 cột) |
| `user_id, status` | Có (2 cột) |
| `user_id, status, order_date` | Có (cả 3 cột) |
| `status` | **Không** |
| `status, order_date` | **Không** |
| `user_id, order_date` | Chỉ dùng được `user_id` (bỏ qua `status`) |

```sql
-- Dùng đủ 3 cột
SELECT * FROM orders
WHERE user_id = 1 AND status = 'PAID' AND order_date >= '2026-01-01';

-- Không dùng được index (thiếu user_id, cột đầu tiên)
SELECT * FROM orders WHERE status = 'PAID';
```

> **Lưu ý:** Thứ tự các điều kiện viết trong `WHERE` **không quan trọng**, MySQL tự sắp xếp lại. Chỉ cần **tập cột** có chứa các cột đầu liên tiếp.

## Thứ tự cột trong Composite Key - Quyết định quan trọng khi thiết kế

Cùng 2 cột nhưng đặt thứ tự khác nhau sẽ phục vụ những truy vấn khác nhau: `(user_id, status)` khác hoàn toàn `(status, user_id)`.

**Quy tắc chọn thứ tự:**

1. **Cột lọc bằng `=` đặt trước, cột lọc khoảng (`>`, `<`, `BETWEEN`) hoặc `ORDER BY` đặt sau.**
2. **Cột xuất hiện trong nhiều truy vấn nhất đặt đầu**, vì leftmost prefix cho phép nó được dùng độc lập.
3. **Trong các cột lọc bằng `=`, cột có độ phân biệt cao hơn đặt trước** (nhiều giá trị khác nhau, loại được nhiều dòng hơn).

**Ví dụ:** Truy vấn phổ biến là "đơn hàng của khách X theo trạng thái".

```sql
-- Tốt: user_id có hàng triệu giá trị, status chỉ vài giá trị
CREATE INDEX idx_user_status ON orders(user_id, status);

-- Kém: status đặt đầu, mỗi giá trị chiếm một phần rất lớn của bảng,
-- và truy vấn chỉ theo user_id sẽ không dùng được index
CREATE INDEX idx_status_user ON orders(status, user_id);
```

> **Ngoại lệ của quy tắc 3:** Nếu có truy vấn thường chỉ lọc theo `status` (ví dụ "tất cả đơn đang chờ xử lý"), thì cần cân nhắc lại. Quy tắc 2 (phục vụ được nhiều truy vấn) thường quan trọng hơn độ phân biệt.

## Composite key có thể phục vụ cho nhiều câu truy vấn khác nhau - không cần tạo quá nhiều index riêng lẻ

Nhờ leftmost prefix, **một** index `(user_id, status, order_date)` phục vụ được cả 3 dạng truy vấn:

```sql
-- 1. Chỉ user_id
SELECT * FROM orders WHERE user_id = 1;

-- 2. user_id + status
SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID';

-- 3. Cả 3 cột, kèm sắp xếp theo ngày không cần sort lại
SELECT * FROM orders
WHERE user_id = 1 AND status = 'PAID'
ORDER BY order_date DESC;
```

Vì vậy **không cần** tạo thêm các index sau, chúng là thừa:

```sql
CREATE INDEX idx_user ON orders(user_id);                  -- thừa
CREATE INDEX idx_user_status ON orders(user_id, status);   -- thừa
```

**Khi nào vẫn cần index riêng:** truy vấn không bắt đầu từ cột đầu của index hiện có, ví dụ lọc riêng theo `status` hoặc `order_date`.

**Kiểm tra và dọn index thừa:**

```sql
SHOW INDEX FROM orders;

-- Xem MySQL dùng index nào, dùng bao nhiêu cột
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'PAID';
```

Trong kết quả `EXPLAIN`, cột `key` cho biết index được chọn, cột `key_len` cho biết **đã dùng bao nhiêu cột** của index (độ dài càng lớn thì dùng càng nhiều cột).

## Trường hợp đặc biệt: Composite index và điều kiện RANGE (>, <, BETWEEN)

Khi một cột trong index bị lọc bằng **khoảng**, các cột đứng **sau** nó **không còn dùng được để tìm kiếm** (chỉ lọc thêm sau khi đã đọc). Lý do: trong một khoảng giá trị, các cột sau không còn được sắp xếp liên tục.

**Ví dụ:** Index `(user_id, order_date, status)`.

```sql
CREATE INDEX idx_user_date_status ON orders(user_id, order_date, status);

SELECT * FROM orders
WHERE user_id = 1
  AND order_date >= '2026-01-01'      -- cột khoảng
  AND status = 'PAID';                -- đứng sau cột khoảng
```

- `user_id = 1` và `order_date >= ...` được dùng để tìm kiếm.
- `status = 'PAID'` **không giúp thu hẹp vùng tìm**, MySQL phải đọc hết các dòng trong khoảng ngày rồi mới lọc `status`.

**Cách sửa: đặt cột khoảng ở cuối.**

```sql
-- Cột = đứng trước, cột khoảng đứng cuối
CREATE INDEX idx_user_status_date ON orders(user_id, status, order_date);
```

Với index này, cả 3 điều kiện đều được dùng để tìm kiếm.

**Ảnh hưởng đến `ORDER BY`:** nếu có cột khoảng ở giữa, việc sắp xếp theo cột phía sau cũng không dùng được index, MySQL sẽ phải sắp xếp lại (`Using filesort` trong `EXPLAIN`).

## Tổng kết

| Quy tắc | Ghi nhớ |
|---|---|
| Cấu trúc | Sắp theo cột 1, rồi cột 2 trong cùng cột 1, rồi cột 3... |
| Leftmost prefix | Phải dùng các cột liên tiếp từ trái sang, không bỏ cột giữa |
| Thứ tự cột | Cột `=` trước, cột khoảng / `ORDER BY` sau, cột dùng nhiều nhất đặt đầu |
| Tiết kiệm index | 1 composite thay cho nhiều index riêng lẻ, tránh index thừa |
| Cột khoảng | Các cột sau nó không dùng để tìm kiếm được, nên đặt cuối |

**Các lỗi hay gặp:**
- Tạo index `(a, b)` rồi truy vấn chỉ theo `b`.
- Đặt cột khoảng ở giữa làm các cột sau mất tác dụng.
- Tạo thêm index riêng cho cột đầu trong khi composite index đã phục vụ được.
- Thêm quá nhiều cột vào composite index, làm index nặng và ghi chậm.

**Cách nhớ nhanh:** Composite index giống **danh bạ sắp theo (họ, tên, ngày sinh)**. Tra theo họ thì nhanh, tra theo họ + tên cũng nhanh, nhưng tra riêng theo tên thì phải lật cả cuốn.
