# Data type

> **Data type (kiểu dữ liệu)** quy định mỗi cột trong bảng được phép chứa loại dữ liệu nào (số, chữ, ngày giờ...) và chiếm bao nhiêu dung lượng. Khi tạo bảng, mỗi cột bắt buộc phải chọn một kiểu dữ liệu.

## Tại sao chọn đúng kiểu dữ liệu lại quan trọng

- **Tiết kiệm bộ nhớ:** Mỗi kiểu chiếm dung lượng khác nhau. Lưu số tuổi bằng `TINYINT` (1 byte) thay vì `BIGINT` (8 byte) sẽ nhẹ hơn nhiều khi bảng có hàng triệu dòng.
- **Đảm bảo tính đúng đắn:** Cột kiểu số không cho phép nhập chữ, cột kiểu ngày không cho phép nhập ngày 31/02. Database tự chặn dữ liệu sai ngay từ đầu.
- **Tăng tốc độ truy vấn:** Dữ liệu nhỏ gọn và đúng kiểu thì so sánh, sắp xếp và đánh index nhanh hơn. Ví dụ so sánh số nhanh hơn so sánh chuỗi.

## Các kiểu dữ liệu quan trọng

### 1. Kiểu số nguyên - INT | TINYINT | BIGINT

Dùng cho số không có phần thập phân: id, số lượng, tuổi...

| Kiểu | Dung lượng | Phạm vi (có dấu) | Dùng khi |
|---|---|---|---|
| TINYINT | 1 byte | -128 đến 127 | Số nhỏ: tuổi, trạng thái, cờ 0/1 |
| INT | 4 byte | khoảng ±2,1 tỷ | Phổ biến nhất: id, số lượng |
| BIGINT | 8 byte | khoảng ±9,2 tỷ tỷ | Số rất lớn: id của bảng hàng trăm triệu dòng |

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    quantity INT NOT NULL,
    rating TINYINT
);
```

> **Mẹo:** Thêm `UNSIGNED` nếu số không bao giờ âm, phạm vi sẽ gấp đôi. Ví dụ `TINYINT UNSIGNED` là 0 đến 255. Trong Spring Boot, `INT` ứng với `Integer`, `BIGINT` ứng với `Long`.

### 2. Kiểu tiền tệ - DECIMAL

Dùng cho **tiền và mọi số cần chính xác tuyệt đối**. `DECIMAL(M, D)` có `M` là tổng số chữ số, `D` là số chữ số sau dấu phẩy.

```sql
price DECIMAL(10, 2)   -- tối đa 99.999.999,99
```

> **Không dùng `FLOAT` hay `DOUBLE` cho tiền.** Hai kiểu này lưu số gần đúng nên có thể sai số nhỏ (ví dụ 0.1 + 0.2 ra 0.30000000000000004). Trong Spring Boot, `DECIMAL` ứng với `BigDecimal`.

### 3. Kiểu chuỗi - VARCHAR | NVARCHAR | TEXT

| Kiểu | Đặc điểm | Dùng khi |
|---|---|---|
| VARCHAR(n) | Chuỗi độ dài thay đổi, tối đa `n` ký tự | Tên, email, số điện thoại, địa chỉ |
| NVARCHAR(n) | Giống VARCHAR nhưng dành cho ký tự Unicode | Có trong SQL Server, ít cần trong MySQL |
| TEXT | Chuỗi dài, không cần khai báo độ dài | Mô tả sản phẩm, nội dung bài viết |

```sql
name VARCHAR(100) NOT NULL,
description TEXT
```

> **Lưu ý cho MySQL:** MySQL hỗ trợ tiếng Việt có dấu bằng cách đặt bảng mã `utf8mb4` cho database hoặc bảng, nên chỉ cần dùng `VARCHAR`. `NVARCHAR` là kiểu của SQL Server, bạn sẽ gặp khi đọc tài liệu chung về SQL.
>
> Chọn `n` vừa đủ dùng (tên: 100, email: 255), không nên đặt quá lớn. Cột `TEXT` không nên dùng để lọc hay tìm kiếm thường xuyên vì chậm. Trong Spring Boot, chuỗi ứng với `String`.

### 4. Kiểu ngày giờ - DATETIME | DATE | TIMESTAMP

| Kiểu | Lưu gì | Định dạng | Đặc điểm |
|---|---|---|---|
| DATE | Chỉ ngày | `2026-11-23` | Ngày sinh, ngày giao hàng |
| DATETIME | Ngày + giờ | `2026-11-23 14:30:00` | Lưu đúng giá trị nhập vào, không phụ thuộc múi giờ |
| TIMESTAMP | Ngày + giờ | `2026-11-23 14:30:00` | Tự đổi theo múi giờ của kết nối, chỉ lưu được đến năm 2038 |

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    delivery_date DATE,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP
);
```

> **Cách chọn:** Dùng `DATETIME` cho thời điểm nghiệp vụ như ngày đặt hàng. Dùng `TIMESTAMP` cho cột theo dõi thay đổi như `created_at`, `updated_at`. Trong Spring Boot, `DATE` ứng với `LocalDate`, `DATETIME` và `TIMESTAMP` ứng với `LocalDateTime`.

### 5. Kiểu liệt kê - ENUM

Cột chỉ được nhận **một giá trị trong danh sách cố định** khai báo sẵn. Phù hợp cho các trạng thái ít thay đổi.

```sql
status ENUM('PENDING', 'PAID', 'SHIPPED', 'CANCELLED') DEFAULT 'PENDING'
```

Nhập giá trị ngoài danh sách (ví dụ `'ABC'`) sẽ bị báo lỗi.

> **Ưu điểm:** Gọn, dễ đọc, dữ liệu luôn hợp lệ.
> **Nhược điểm:** Muốn thêm hoặc đổi giá trị phải dùng `ALTER TABLE`. Nếu trạng thái hay thay đổi, nên tạo bảng riêng thay vì dùng ENUM.
>
> Trong Spring Boot, ENUM ứng với `enum` của Java. Nhớ dùng `@Enumerated(EnumType.STRING)` để lưu chữ, tránh lưu số thứ tự dễ sai khi đổi thứ tự.

### 6. Kiểu Boolean

Dùng cho giá trị **đúng/sai**: sản phẩm còn bán không, tài khoản đã kích hoạt chưa...

```sql
is_active BOOLEAN DEFAULT TRUE
```

> **Lưu ý:** MySQL không có kiểu boolean thật. `BOOLEAN` thực chất là `TINYINT(1)`, trong đó `0` là false và `1` (hoặc số khác 0) là true. Bạn có thể viết `TRUE`/`FALSE` hoặc `1`/`0` đều được. Trong Spring Boot, ứng với `boolean` hoặc `Boolean`.

## Tổng kết nhanh cho ứng dụng bán hàng

| Dữ liệu | Kiểu nên dùng | Kiểu trong Java |
|---|---|---|
| id | `BIGINT` | `Long` |
| Số lượng tồn kho | `INT` | `Integer` |
| Giá tiền | `DECIMAL(10,2)` | `BigDecimal` |
| Tên sản phẩm | `VARCHAR(100)` | `String` |
| Mô tả | `TEXT` | `String` |
| Ngày đặt hàng | `DATETIME` | `LocalDateTime` |
| Trạng thái đơn hàng | `ENUM` | `enum` |
| Còn bán hay không | `BOOLEAN` | `boolean` |
