# EXPLAIN

> **EXPLAIN** cho biết MySQL **dự định thực thi** một câu truy vấn như thế nào: dùng index nào, đọc bao nhiêu dòng, nối bảng theo thứ tự nào. Đây là công cụ chính để tìm ra vì sao truy vấn chậm và kiểm tra index có được dùng hay không.

Các ví dụ dùng bảng `users` (`id`, `full_name`, `email`) và `orders` (`id`, `user_id`, `status`, `total_amount`, `order_date`) với index `idx_orders_user_status(user_id, status)`.

## Cú pháp và các định dạng hiện có

Đặt `EXPLAIN` trước câu lệnh cần phân tích (`SELECT`, `INSERT`, `UPDATE`, `DELETE`). Có 3 định dạng.

### 1. Dạng bảng (mặc định)

```sql
EXPLAIN
SELECT * FROM orders WHERE user_id = 10 AND status = 'PAID';
```

Kết quả là một bảng, mỗi dòng ứng với một bảng được đọc. Chỉ cần nhìn nhanh 3 cột:

| Cột | Ý nghĩa | Cần để ý |
|---|---|---|
| `type` | Cách truy cập bảng | `ALL` là quét toàn bộ bảng (xấu). `const`, `ref`, `range` là tốt |
| `key` | Index thực sự được dùng | `NULL` nghĩa là không dùng index |
| `Extra` | Thông tin bổ sung | `Using filesort`, `Using temporary` là dấu hiệu xấu. `Using index` là tốt |

Dạng này gọn, phù hợp để **kiểm tra nhanh** truy vấn có dùng index hay không.

### 2. Dạng cây (TREE)

```sql
EXPLAIN FORMAT=TREE
SELECT * FROM orders WHERE user_id = 10 AND status = 'PAID';
```

```
-> Index lookup on orders using idx_orders_user_status (user_id=10, status='PAID')  (cost=1.05 rows=3)
```

Hiển thị **các bước thực thi dạng lồng nhau**, dễ đọc và gần với cách MySQL chạy thật. **Đọc từ trong ra ngoài, từ trên xuống dưới**: bước thụt vào sâu nhất chạy trước, kết quả đưa lên bước bên ngoài. Cần MySQL 8.0.16 trở lên. Đây là dạng nên dùng nhiều nhất.

### 3. Dạng JSON

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM orders WHERE user_id = 10 AND status = 'PAID';
```

Cho **thông tin đầy đủ nhất** nhưng rất dài và khó đọc bằng mắt. Thường dùng khi cần đưa kết quả vào công cụ phân tích hoặc công cụ trực quan hóa, còn đọc hằng ngày thì dạng cây tiện hơn.

## Các từ khóa cần nhận diện với dạng cây

| Từ khóa | Ý nghĩa | Đánh giá |
|---|---|---|
| `Table scan` | Quét toàn bộ bảng | Xấu trên bảng lớn |
| `Index lookup` | Tìm theo index, ra nhiều dòng | Tốt |
| `Single-row index lookup` | Tìm 1 dòng bằng PK/unique | Rất tốt |
| `Index range scan` | Quét một khoảng trong index | Tốt |
| `Index scan` | Quét toàn bộ index | Trung bình |
| `Covering index lookup` | Lấy dữ liệu chỉ từ index, không đọc bảng | Rất tốt |
| `Filter` | Lọc dòng theo điều kiện sau khi đọc | Bình thường, nhưng nếu đứng trên `Table scan` là dấu hiệu thiếu index |
| `Nested loop inner join` | Với mỗi dòng bảng ngoài, tìm dòng khớp ở bảng trong | Bình thường, hiệu quả khi bảng trong có index |
| `Hash join` | Nối bằng bảng băm khi không có index để nối | Nên xem xét thêm index cho cột `JOIN` |
| `Sort` | Sắp xếp lại kết quả | Tốn kém nếu nhiều dòng |
| `Aggregate` / `Group aggregate` | Thực hiện `GROUP BY`, hàm tổng hợp | Bình thường |
| `Temporary table` | Dùng bảng tạm | Cần chú ý |

**Ví dụ xấu** (thiếu index trên `total_amount`):

```sql
EXPLAIN FORMAT=TREE
SELECT * FROM orders WHERE total_amount > 500000;
```

```
-> Filter: (orders.total_amount > 500000)  (cost=10234.50 rows=33333)
    -> Table scan on orders  (cost=10234.50 rows=100000)
```

Đọc từ trong ra: quét toàn bộ 100.000 dòng (`Table scan`), rồi mới lọc (`Filter`). Cặp `Filter` đứng trên `Table scan` là dấu hiệu điển hình cần thêm index.

**Ví dụ tốt** (JOIN dùng index ở cả hai bảng):

```sql
EXPLAIN FORMAT=TREE
SELECT u.full_name, o.id
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.id = 10;
```

```
-> Nested loop inner join  (cost=2.10 rows=3)
    -> Single-row index lookup on u using PRIMARY (id=10)  (cost=0.35 rows=1)
    -> Index lookup on o using idx_orders_user_status (user_id=10)  (cost=1.75 rows=3)
```

Tìm 1 người dùng bằng khóa chính, rồi dùng index để lấy các đơn hàng của họ. Không có `Table scan` nào.

## Đọc 2 con số `cost` và `rows` trong mỗi dòng

Mỗi bước trong dạng cây đều kèm `(cost=... rows=...)`.

| Con số | Ý nghĩa |
|---|---|
| `cost` | **Chi phí ước tính** của bước đó (công sức đọc đĩa, xử lý CPU). Không có đơn vị cụ thể, chỉ dùng để **so sánh tương đối** |
| `rows` | **Số dòng ước tính** bước đó sẽ trả về |

**Cách đọc đúng:**
- `cost` của bước bên ngoài **đã bao gồm** cost của các bước bên trong, nên dòng trên cùng là **tổng chi phí** của cả truy vấn.
- Dùng `cost` để **so sánh hai cách viết** của cùng một truy vấn hoặc **trước và sau khi thêm index**. Con số nhỏ hơn là tốt hơn (ở ví dụ trên giảm từ khoảng 10.234 xuống còn khoảng 1).
- `rows` lớn bất thường ở một bước là chỗ cần tối ưu.
- Trong `JOIN`, số dòng thực tế của bảng trong là `rows` nhân với số lần lặp (số dòng của bảng ngoài).

> **Lưu ý:** Cả hai đều là **số ước tính** dựa trên thống kê của bảng. Thống kê cũ có thể làm ước tính sai lệch nhiều. Muốn biết con số thật thì dùng `EXPLAIN ANALYZE`.

## Cách xem chi tiết thời gian thực tế: EXPLAIN ANALYZE

`EXPLAIN ANALYZE` **chạy thật** câu truy vấn, rồi trả về kế hoạch thực thi kèm **số liệu đo được**. Cần MySQL 8.0.18 trở lên, kết quả ở dạng cây.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 10 AND status = 'PAID';
```

```
-> Index lookup on orders using idx_orders_user_status (user_id=10, status='PAID')
   (cost=1.05 rows=3) (actual time=0.045..0.052 rows=3 loops=1)
```

**Cách đọc phần `actual`:**

| Thành phần | Ý nghĩa |
|---|---|
| `actual time=0.045..0.052` | Thời gian (ms): **0.045** để lấy dòng đầu tiên, **0.052** để lấy xong tất cả |
| `rows=3` | **Số dòng thật** đã trả về |
| `loops=1` | Bước này được thực hiện **bao nhiêu lần** |

**Cách đánh giá:**
- **Tổng thời gian thực của một bước = thời gian × loops.** Bước nằm trong vòng lặp `JOIN` có thể có thời gian nhỏ nhưng `loops` rất lớn.
- **So sánh `rows` ước tính với `rows` thực tế.** Nếu chênh lệch lớn (ước tính 3, thực tế 30.000), thống kê đã cũ. Chạy `ANALYZE TABLE` để cập nhật.

```sql
ANALYZE TABLE orders;
```

- Tìm **bước có thời gian lớn nhất** để biết nên tối ưu ở đâu.

**Lưu ý quan trọng:**
- `EXPLAIN ANALYZE` **thực sự chạy truy vấn**, nên truy vấn nặng sẽ tốn thời gian và tài nguyên thật. Không chạy tùy tiện trên production.
- Với `UPDATE`/`DELETE`, nên dùng `EXPLAIN` thường, hoặc chạy thử trong transaction rồi `ROLLBACK`.

## Quy trình tối ưu một truy vấn chậm

1. Chạy `EXPLAIN FORMAT=TREE`, tìm `Table scan`, `Sort`, `Temporary table`.
2. Xác định cột nào cần index (`WHERE`, `JOIN`, `ORDER BY`), tạo index phù hợp.
3. Chạy lại `EXPLAIN`, so sánh `cost` và `rows` trước và sau.
4. Dùng `EXPLAIN ANALYZE` để xác nhận thời gian thực tế đã giảm.

## Tổng kết

| Công cụ | Dùng để |
|---|---|
| `EXPLAIN` (dạng bảng) | Kiểm tra nhanh `type`, `key`, `Extra` |
| `EXPLAIN FORMAT=TREE` | Xem các bước thực thi theo thứ tự, dễ đọc nhất |
| `EXPLAIN FORMAT=JSON` | Lấy đầy đủ chi tiết cho công cụ phân tích |
| `EXPLAIN ANALYZE` | Đo **thời gian và số dòng thực tế** |

**Dấu hiệu cần xử lý:** `type = ALL`, `Table scan`, `key = NULL`, `Using filesort`, `Using temporary`, hoặc `rows` ước tính lệch xa thực tế.

**Cách nhớ nhanh:** `EXPLAIN` là "bản kế hoạch" (dự đoán), `EXPLAIN ANALYZE` là "bản báo cáo sau khi chạy" (thực đo). Ưu tiên đọc dạng cây, luôn tìm `Table scan` và so sánh `cost`, `rows` trước và sau khi tối ưu.
