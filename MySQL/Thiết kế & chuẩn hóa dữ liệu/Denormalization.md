# Denormalization

> **Denormalization (phi chuẩn hóa)** là việc **cố ý đưa dữ liệu lặp** trở lại thiết kế (sau khi đã chuẩn hóa) để **đổi lấy tốc độ đọc**. Đây là sự đánh đổi có chủ đích: chấp nhận dữ liệu dư thừa và phức tạp hơn khi ghi, để truy vấn nhanh và đơn giản hơn.

## Tại sao chuẩn hóa quá mức lại đôi khi gây vấn đề hiệu năng?

Chuẩn hóa chia dữ liệu thành nhiều bảng nhỏ, nên muốn lấy một thông tin đầy đủ phải `JOIN` nhiều bảng. Khi dữ liệu lên hàng triệu dòng, mỗi lần `JOIN` đều tốn thời gian và tài nguyên (đọc nhiều bảng, ghép dòng, dùng bộ nhớ tạm). Các truy vấn được gọi liên tục như trang danh sách đơn hàng, báo cáo doanh thu sẽ chậm dần, kéo theo cả ứng dụng chậm. Ngoài ra, truy vấn dài với nhiều `JOIN` cũng khó viết, khó đọc và khó tối ưu.

**Ví dụ:** Hiển thị danh sách đơn hàng kèm tên khách, tên sản phẩm và tổng tiền phải nối 4 bảng:

```sql
SELECT o.id, c.name, p.name, oi.quantity, oi.unit_price
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id;
```

## Các kỹ thuật phi chuẩn hóa phổ biến

### 1. Lưu giá trị trước khi tổng hợp

Thay vì mỗi lần cần đều tính lại bằng `SUM`, `COUNT` trên nhiều dòng, ta **tính sẵn rồi lưu thành một cột**. Khi dữ liệu gốc thay đổi, cập nhật lại cột này. Cách này giúp đọc cực nhanh vì không phải quét và cộng hàng loạt dòng mỗi lần truy vấn. Ngoài ra còn có một lợi ích quan trọng khác: lưu lại giá trị **tại thời điểm xảy ra**, như giá bán lúc đặt hàng, để sau này đổi giá sản phẩm không làm sai đơn cũ.

```sql
ALTER TABLE orders
ADD COLUMN total_amount DECIMAL(12,2) NOT NULL DEFAULT 0;

-- Cập nhật khi đơn hàng thay đổi
UPDATE orders o
SET total_amount = (
    SELECT SUM(quantity * unit_price)
    FROM order_items
    WHERE order_id = o.id
)
WHERE o.id = 1;
```

### 2. Sao chép dữ liệu tham chiếu để tránh JOIN

Sao chép một vài cột hay dùng từ bảng khác sang bảng đang truy vấn, để không cần `JOIN` mỗi lần. Chỉ nên sao chép những cột **nhỏ, hay hiển thị, ít thay đổi** như tên khách, tên sản phẩm.

```sql
ALTER TABLE order_items
ADD COLUMN product_name VARCHAR(100) NOT NULL;
```

Khi hiển thị chi tiết đơn hàng, không cần nối với bảng `products` nữa. Cách này còn giúp giữ lại tên sản phẩm đúng như lúc khách mua, dù sau này sản phẩm đổi tên.

### 3. Gộp bảng cha-con khi luôn truy vấn cùng nhau

Nếu hai bảng quan hệ 1-1 (hoặc dữ liệu con rất ít) và **luôn được đọc cùng nhau**, việc tách riêng chỉ làm tốn thêm một lần `JOIN`. Khi đó có thể gộp thành một bảng, hoặc lưu phần con ngay trong cột của bảng cha (ví dụ dạng `JSON`).

```sql
-- Thay vì bảng user_addresses tách riêng, lưu địa chỉ giao hàng mặc định ngay trong users
ALTER TABLE users
ADD COLUMN default_address VARCHAR(255),
ADD COLUMN default_city VARCHAR(100);
```

### 4. Bảng tổng hợp riêng cho báo cáo

Tạo **bảng riêng chuyên phục vụ báo cáo, thống kê**, chứa số liệu đã tổng hợp sẵn (theo ngày, theo tháng, theo sản phẩm). Báo cáo chỉ đọc bảng này, không quét bảng giao dịch lớn, nên không làm chậm hoạt động bán hàng.

```sql
CREATE TABLE daily_sales_report (
    report_date DATE PRIMARY KEY,
    total_orders INT NOT NULL,
    total_revenue DECIMAL(14,2) NOT NULL
);

-- Chạy định kỳ mỗi đêm để cập nhật số liệu hôm qua
INSERT INTO daily_sales_report (report_date, total_orders, total_revenue)
SELECT DATE(order_date), COUNT(*), SUM(total_amount)
FROM orders
WHERE DATE(order_date) = CURDATE() - INTERVAL 1 DAY
GROUP BY DATE(order_date);
```

## Nguyên tắc quyết định: Khi nào cần phi chuẩn hóa?

### Nguyên tắc 1: Đo lường trước, tối ưu sau

Đừng phi chuẩn hóa chỉ vì "đoán" nó sẽ chậm. Hãy **đo thực tế** trước: bật slow query log, dùng `EXPLAIN` để xem câu lệnh đang chậm ở đâu, đo thời gian phản hồi với lượng dữ liệu gần giống thật. Rất nhiều trường hợp chậm chỉ cần **thêm index** hoặc **viết lại câu truy vấn** là xong, không cần đụng đến thiết kế bảng. Chỉ khi đã thử các cách đó mà vẫn là điểm nghẽn thì mới cân nhắc phi chuẩn hóa.

```sql
EXPLAIN
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

### Nguyên tắc 2: Tỷ lệ đọc/ghi là yếu tố quyết định

Phi chuẩn hóa làm **đọc nhanh hơn nhưng ghi phức tạp hơn** (phải cập nhật nhiều nơi). Vì vậy:

- **Đọc nhiều, ghi ít** (danh sách sản phẩm, báo cáo, lịch sử đơn hàng): phù hợp phi chuẩn hóa.
- **Ghi nhiều, thay đổi liên tục** (số lượng tồn kho, trạng thái đơn đang xử lý): nên giữ chuẩn hóa để tránh lệch dữ liệu.

### Nguyên tắc 3: Luôn có kế hoạch giữ đồng bộ

Dữ liệu lặp ở nhiều nơi thì sẽ **có nguy cơ lệch nhau**. Trước khi phi chuẩn hóa phải trả lời được: ai cập nhật, cập nhật khi nào, nếu lệch thì sửa thế nào. Các cách phổ biến:

- Cập nhật trong **cùng một transaction** ở tầng ứng dụng (Spring Boot `@Transactional`).
- Dùng **trigger** của database.
- Chạy **job định kỳ** (mỗi đêm) tính lại, phù hợp với bảng báo cáo.

Không có kế hoạch này thì không nên phi chuẩn hóa.

### Nguyên tắc 4: Ưu tiên chuẩn hóa trước, phi chuẩn hóa là ngoại lệ có lý do rõ ràng

Phi chuẩn hóa không phải là cách thiết kế mặc định, mà là một **ngoại lệ có lý do** và nên được ghi chú lại (vì sao làm, cột nào là bản sao, đồng bộ bằng cách nào).

**Quy trình đúng trong thực tế:**

1. **Thiết kế đạt 3NF** với dữ liệu sạch, không lặp.
2. **Xây dựng ứng dụng** hoạt động đúng chức năng.
3. **Đo lường hiệu năng** thực tế với dữ liệu và lượng truy cập gần như thật.
4. **Chỉ phi chuẩn hóa những điểm nghẽn** đã xác định rõ, từng chỗ một, và kiểm tra lại hiệu quả.

## Tổng kết

| Kỹ thuật | Lợi ích | Cái giá phải trả |
|---|---|---|
| Lưu giá trị đã tổng hợp | Không phải tính lại mỗi lần, giữ được giá trị lúc mua | Phải cập nhật khi dữ liệu gốc đổi |
| Sao chép dữ liệu tham chiếu | Bớt `JOIN` | Dữ liệu lặp, có thể lệch |
| Gộp bảng cha-con | Bớt `JOIN`, truy vấn đơn giản | Bảng to hơn, cột dư |
| Bảng tổng hợp cho báo cáo | Báo cáo nhanh, không ảnh hưởng bảng giao dịch | Thêm bảng, cần job cập nhật |

**Cách nhớ nhanh:** Chuẩn hóa ưu tiên **ghi đúng, ghi gọn**; phi chuẩn hóa ưu tiên **đọc nhanh**. Hãy chuẩn hóa trước, đo lường rồi mới phi chuẩn hóa đúng chỗ cần.
