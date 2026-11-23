## Bảng (Table/Relation)

- **Khái niệm:** Nơi lưu trữ dữ liệu của **một đối tượng** trong hệ thống, được tổ chức dưới dạng hàng và cột, giống một bảng tính Excel. Ví dụ: bảng `products` lưu sản phẩm, bảng `users` lưu người dùng.
- **Đặc điểm:**
  - Mỗi bảng có một **tên duy nhất** trong database và chỉ lưu một loại đối tượng.
  - Cấu trúc bảng (tên cột, kiểu dữ liệu) được định nghĩa trước bằng lệnh DDL.
  - Thường có một **khóa chính** để phân biệt các dòng và có thể liên kết với bảng khác bằng khóa ngoại.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    quantity INT DEFAULT 0
);
```

## Hàng (Row/Record/Tuple)

- **Khái niệm:** Một **bản ghi dữ liệu hoàn chỉnh** của một đối tượng, nằm ngang trong bảng. Ví dụ: một dòng là thông tin của một sản phẩm cụ thể.
- **Đặc điểm:**
  - Mỗi hàng đại diện cho **một đối tượng** (một sản phẩm, một người dùng, một đơn hàng).
  - Các hàng trong cùng bảng có **cùng cấu trúc**, tức là cùng các cột.
  - Mỗi hàng được phân biệt với hàng khác bằng **khóa chính**, và số lượng hàng có thể tăng giảm tùy ý khi thêm hoặc xóa dữ liệu.

```sql
INSERT INTO products (name, price, quantity)
VALUES ('Áo thun', 150000, 20);
```

Câu lệnh trên thêm **1 hàng** mới vào bảng.

## Cột (Column/Field/Attribute)

- **Khái niệm:** Một **thuộc tính** mô tả đối tượng, nằm dọc trong bảng. Ví dụ: `name`, `price`, `quantity` là các thuộc tính của sản phẩm.
- **Đặc điểm:**
  - Mỗi cột có **tên** và **kiểu dữ liệu** cố định (số, chuỗi, ngày giờ...), mọi giá trị trong cột phải thuộc kiểu đó.
  - Cột có thể gắn thêm **ràng buộc** như `NOT NULL`, `UNIQUE`, `DEFAULT`.
  - Số lượng và tên cột do người thiết kế quyết định, thay đổi được bằng `ALTER TABLE`.

```sql
SELECT name, price
FROM products;
```

Câu lệnh trên chỉ lấy ra **2 cột** `name` và `price`.

## Minh họa tổng hợp

Bảng `products` với dữ liệu mẫu:

| id | name | price | quantity |
|---|---|---|---|
| 1 | Áo thun | 150000 | 20 |
| 2 | Quần jean | 350000 | 15 |
| 3 | Giày thể thao | 800000 | 10 |

- Toàn bộ bảng trên là **1 Table** (`products`).
- Mỗi dòng ngang, ví dụ `2 | Quần jean | 350000 | 15`, là **1 Row**.
- Mỗi cột dọc, ví dụ `price`, là **1 Column**.
- Giao điểm của một hàng và một cột, ví dụ `350000`, là **1 giá trị (cell)**.

**Cách nhớ nhanh:** Table là cả bảng Excel, Row là một dòng thông tin của một đối tượng, Column là một loại thông tin chung cho mọi đối tượng.

Vậy là bạn đã có đủ nội dung cho giai đoạn 1: DDL, DML, Data type, Key và Table - Row - Column. Khi cần mình có thể làm tiếp giai đoạn 2 của lộ trình hoặc tổng hợp thành bài tập thực hành để bạn dùng hướng dẫn người khác.
