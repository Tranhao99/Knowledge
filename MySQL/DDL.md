# DDL

> **DDL (Data Definition Language - Ngôn ngữ định nghĩa dữ liệu)** là nhóm lệnh SQL dùng để **tạo, sửa, xóa cấu trúc** của database: database, bảng (table), cột (column), index... DDL chỉ quan tâm đến "cái khung" chứa dữ liệu, không quan tâm đến dữ liệu bên trong.

Hiểu đơn giản: nếu database là một tòa nhà, DDL là việc xây nhà, sửa phòng, đập phòng. Còn việc cất đồ vào phòng (thêm, sửa, xóa dữ liệu) là của DML.

## Các lệnh cơ bản

### 1. CREATE DATABASE - Tạo database

Tạo một "kho" mới chứa các bảng.

```sql
CREATE DATABASE shop_db;
```

Nên thêm `IF NOT EXISTS` để không bị lỗi nếu database đã tồn tại:

```sql
CREATE DATABASE IF NOT EXISTS shop_db;
```

### 2. USE - Chọn database để làm việc

Sau khi tạo xong, cần chọn database trước khi tạo bảng.

```sql
USE shop_db;
```

### 3. CREATE TABLE - Tạo bảng

Tạo bảng mới, khai báo tên cột, kiểu dữ liệu và các ràng buộc.

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    quantity INT DEFAULT 0
);
```

### 4. ALTER TABLE - Sửa cấu trúc bảng

Dùng khi bảng đã có nhưng cần thay đổi. Có nhiều kiểu sửa:

**Thêm cột (ADD COLUMN):**

```sql
ALTER TABLE products
ADD COLUMN description TEXT;
```

**Sửa kiểu dữ liệu của cột (MODIFY COLUMN):**

```sql
ALTER TABLE products
MODIFY COLUMN name VARCHAR(200) NOT NULL;
```

**Đổi tên cột (RENAME COLUMN):**

```sql
ALTER TABLE products
RENAME COLUMN name TO product_name;
```

**Đổi tên và sửa kiểu dữ liệu cùng lúc (CHANGE COLUMN):**

CHANGE COLUMN làm được cả hai việc trong một lệnh, nhưng phải khai báo lại đầy đủ: tên cũ, tên mới, kiểu dữ liệu và các ràng buộc. Nếu chỉ muốn sửa kiểu thì dùng MODIFY, chỉ muốn đổi tên thì dùng RENAME COLUMN cho gọn.

```sql
ALTER TABLE products
CHANGE COLUMN name product_name VARCHAR(200) NOT NULL;
```

**Xóa cột (DROP COLUMN):**

```sql
ALTER TABLE products
DROP COLUMN description;
```

### 5. RENAME TABLE - Đổi tên bảng

```sql
RENAME TABLE products TO items;
```

### 6. DROP TABLE - Xóa bảng

Xóa luôn cả bảng (cấu trúc lẫn dữ liệu). Sau lệnh này, bảng không còn tồn tại.

```sql
DROP TABLE products;
```

Nên thêm `IF EXISTS` để tránh lỗi khi bảng không có:

```sql
DROP TABLE IF EXISTS products;
```

### 7. TRUNCATE TABLE - Xóa sạch dữ liệu trong bảng

Giữ lại cấu trúc bảng, chỉ xóa toàn bộ các dòng dữ liệu.

```sql
TRUNCATE TABLE products;
```

### 8. DROP DATABASE - Xóa database

Xóa toàn bộ database cùng tất cả bảng bên trong. **Rất nguy hiểm**, cần cẩn thận.

```sql
DROP DATABASE shop_db;
```

> **Lưu ý:** Các lệnh DDL trong MySQL khi chạy xong sẽ tự động lưu (auto commit), **không thể hoàn tác (rollback)**. Hãy sao lưu dữ liệu trước khi chạy `DROP` hoặc `TRUNCATE` trên dữ liệu quan trọng.

## So sánh DROP TABLE | TRUNCATE TABLE | DELETE

Cả ba lệnh đều liên quan đến việc "xóa", nhưng xóa những thứ khác nhau.

| Tiêu chí | DROP TABLE | TRUNCATE TABLE | DELETE |
|---|---|---|---|
| Thuộc nhóm | DDL | DDL | DML |
| Xóa cái gì | Cả bảng (cấu trúc + dữ liệu) | Toàn bộ dữ liệu, giữ cấu trúc bảng | Các dòng dữ liệu |
| Xóa theo điều kiện (WHERE) | Không | Không | Có |
| Bảng còn tồn tại sau khi xóa | Không | Có | Có |
| Reset AUTO_INCREMENT về 1 | Không áp dụng (bảng mất) | Có | Không (id tiếp tục tăng) |
| Hoàn tác (rollback) | Không | Không | Có, nếu dùng trong transaction |
| Tốc độ | Nhanh | Nhanh | Chậm hơn khi nhiều dòng |

**Ví dụ:**

```sql
-- Xóa 1 sản phẩm có id = 5 (dùng DELETE)
DELETE FROM products
WHERE id = 5;

-- Xóa sạch sản phẩm, giữ lại bảng, id đếm lại từ 1 (dùng TRUNCATE)
TRUNCATE TABLE products;

-- Xóa luôn bảng products khỏi database (dùng DROP)
DROP TABLE products;
```

**Khi nào dùng cái nào?**

- **DELETE:** khi chỉ muốn xóa một số dòng cụ thể, ví dụ xóa đơn hàng bị hủy.
- **TRUNCATE:** khi muốn dọn sạch dữ liệu để làm lại từ đầu, ví dụ xóa dữ liệu test.
- **DROP:** khi bảng không còn cần dùng nữa, muốn xóa hẳn khỏi database.

**Cách nhớ nhanh:** DELETE là "nhặt từng món đồ ra", TRUNCATE là "dọn sạch phòng nhưng giữ lại phòng", DROP là "đập luôn căn phòng".
