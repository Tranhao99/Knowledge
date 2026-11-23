# Key

> **Key (khóa)** là một hoặc nhiều cột dùng để **nhận diện duy nhất một dòng** trong bảng và **tạo liên kết giữa các bảng**. Key giúp dữ liệu không bị trùng lặp, không bị sai lệch và truy vấn nhanh hơn.

Ví dụ minh họa dùng bảng `users` (người dùng) gồm các cột: `id`, `email`, `phone`, `full_name`. Giả sử mỗi `id`, `email`, `phone` đều không trùng nhau, còn `full_name` có thể trùng.

## Siêu khóa (Super key)

- **Định nghĩa:** Tập hợp một hoặc nhiều cột mà giá trị của chúng **xác định duy nhất** một dòng trong bảng.
- **Đặc điểm:**
  - Một bảng có thể có rất nhiều siêu khóa.
  - Có thể chứa cả những cột thừa (không cần thiết để phân biệt dòng).
  - Mọi khóa khác đều bắt nguồn từ siêu khóa.
- **Ví dụ:** Các tập sau đều là siêu khóa của `users`: `{id}`, `{email}`, `{id, email}`, `{id, full_name}`. Riêng `{full_name}` không phải vì tên có thể trùng.

## Khóa dự tuyển (Candidate key)

- **Định nghĩa:** Siêu khóa **tối thiểu**, tức là bỏ bớt bất kỳ cột nào thì không còn xác định duy nhất được nữa.
- **Đặc điểm:**
  - Không chứa cột thừa.
  - Một bảng có thể có nhiều khóa dự tuyển.
  - Là "danh sách ứng viên" để chọn ra khóa chính.
- **Ví dụ:** `{id}`, `{email}`, `{phone}` là khóa dự tuyển. `{id, email}` thì không, vì bỏ `email` đi vẫn đủ xác định bằng `id`.

## Khóa chính (Primary Key)

- **Định nghĩa:** Khóa dự tuyển **được chọn** làm khóa đại diện chính thức cho bảng.
- **Đặc điểm:**
  - Mỗi bảng chỉ có **1** khóa chính.
  - Giá trị **không trùng** và **không được NULL**.
  - Tự động được đánh index nên truy vấn theo khóa chính rất nhanh.
- **Ví dụ:**

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) UNIQUE,
    full_name VARCHAR(100) NOT NULL
);
```

Ở đây `id` là khóa chính, còn `email` và `phone` là các khóa dự tuyển còn lại (được giữ không trùng bằng `UNIQUE`).

## Khóa tổng hợp (Composite key)

- **Định nghĩa:** Khóa chính được tạo từ **2 cột trở lên** kết hợp lại, vì từng cột riêng lẻ chưa đủ để phân biệt các dòng.
- **Đặc điểm:**
  - Từng cột có thể trùng, nhưng **cả cặp** thì không được trùng.
  - Hay gặp ở bảng trung gian (quan hệ nhiều-nhiều).
  - Không cột nào trong khóa được NULL.
- **Ví dụ:** Bảng chi tiết đơn hàng: một đơn có nhiều sản phẩm, một sản phẩm nằm trong nhiều đơn, nhưng mỗi sản phẩm chỉ xuất hiện một lần trong cùng một đơn.

```sql
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

## Khóa ngoại (Foreign key)

- **Định nghĩa:** Cột trong bảng này dùng để **tham chiếu đến khóa chính** của bảng khác, tạo mối liên kết giữa hai bảng.
- **Đặc điểm:**
  - Giá trị phải **tồn tại** ở bảng được tham chiếu (hoặc là NULL).
  - Giá trị có thể **trùng nhau** (nhiều đơn hàng cùng thuộc một người dùng).
  - Đảm bảo **toàn vẹn dữ liệu**: không thể có đơn hàng của một người dùng không tồn tại.
- **Ví dụ:**

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

Hành vi khi xóa/sửa dòng ở bảng cha:

| Tùy chọn | Ý nghĩa |
|---|---|
| `RESTRICT` | Không cho xóa/sửa nếu còn dòng con tham chiếu (mặc định) |
| `CASCADE` | Xóa/sửa dòng cha thì dòng con tự động xóa/sửa theo |
| `SET NULL` | Xóa dòng cha thì cột khóa ngoại của dòng con thành NULL |

## Khóa tự nhiên (Natural key)

- **Định nghĩa:** Khóa lấy từ **dữ liệu có ý nghĩa thực tế** trong nghiệp vụ.
- **Đặc điểm:**
  - Người dùng nhìn vào hiểu ngay (email, số CCCD, mã ISBN của sách, biển số xe).
  - Có nguy cơ **thay đổi** theo thời gian (đổi email, đổi số điện thoại).
  - Thường dài hơn số nguyên nên tốn bộ nhớ và chậm hơn khi liên kết bảng.
- **Ví dụ:**

```sql
CREATE TABLE books (
    isbn VARCHAR(13) PRIMARY KEY,
    title VARCHAR(200) NOT NULL
);
```

`isbn` là khóa tự nhiên, được dùng trực tiếp làm khóa chính.

## Khóa thay thế (Surrogate key)

- **Định nghĩa:** Khóa do hệ thống **tự sinh ra**, không mang ý nghĩa nghiệp vụ, chỉ dùng để phân biệt các dòng.
- **Đặc điểm:**
  - Thường là số tự tăng (`AUTO_INCREMENT`) hoặc `UUID`.
  - **Không bao giờ thay đổi**, nên các bảng liên kết luôn ổn định.
  - Ngắn gọn, truy vấn và JOIN nhanh.
- **Ví dụ:**

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL
);
```

`id` là khóa thay thế (làm khóa chính), còn `sku` là khóa tự nhiên được giữ không trùng bằng `UNIQUE`.

> **Thực tế khi làm Spring Boot:** Cách phổ biến nhất là dùng **khóa thay thế làm khóa chính** (`@GeneratedValue`), đồng thời đặt `UNIQUE` cho khóa tự nhiên như email, sku.

## Khóa ngoại tổng hợp (Composite foreign key)

- **Định nghĩa:** Khóa ngoại gồm **2 cột trở lên**, dùng để tham chiếu đến một **khóa chính tổng hợp** (hoặc khóa duy nhất tổng hợp) của bảng khác.
- **Đặc điểm:**
  - Số cột và kiểu dữ liệu phải **khớp từng cột** với khóa được tham chiếu.
  - Các cột phải được khai báo **cùng một lúc** trong một ràng buộc.
  - Chỉ cần dùng khi bảng cha có khóa tổng hợp.
- **Ví dụ:** Bảng biến thể sản phẩm có khóa chính là cặp (`product_id`, `size`). Bảng giỏ hàng phải tham chiếu đúng cả cặp này.

```sql
CREATE TABLE product_variants (
    product_id BIGINT,
    size VARCHAR(10),
    stock INT NOT NULL,
    PRIMARY KEY (product_id, size)
);

CREATE TABLE cart_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    size VARCHAR(10) NOT NULL,
    quantity INT NOT NULL,
    CONSTRAINT fk_cart_variant
        FOREIGN KEY (product_id, size)
        REFERENCES product_variants(product_id, size)
);
```

## Câu lệnh kiểm tra các FOREIGN KEY đang tồn tại trong database

**Xem tất cả khóa ngoại của một database:**

```sql
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'shop_db'
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

**Xem khóa ngoại của một bảng cụ thể** (thêm điều kiện tên bảng):

```sql
SELECT
    CONSTRAINT_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'shop_db'
  AND TABLE_NAME = 'orders'
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

**Xem kèm hành vi ON DELETE / ON UPDATE:**

```sql
SELECT
    CONSTRAINT_NAME,
    TABLE_NAME,
    REFERENCED_TABLE_NAME,
    DELETE_RULE,
    UPDATE_RULE
FROM information_schema.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'shop_db';
```

**Cách nhanh nhất để xem toàn bộ cấu trúc một bảng (gồm cả khóa ngoại):**

```sql
SHOW CREATE TABLE orders;
```

**Xóa một khóa ngoại** khi cần (dùng đúng tên ở cột `CONSTRAINT_NAME`):

```sql
ALTER TABLE orders
DROP FOREIGN KEY fk_orders_user;
```
