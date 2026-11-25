# Phân quyền User trong MySQL

- **Không dùng `root` cho ứng dụng.** `root` có toàn quyền (xóa database, tạo user, đọc mọi dữ liệu). Nếu ứng dụng bị tấn công (ví dụ SQL Injection), kẻ tấn công cũng có toàn quyền đó.
- **Nguyên tắc đặc quyền tối thiểu (least privilege):** mỗi user chỉ được cấp **đúng những quyền cần thiết** trên **đúng những đối tượng cần thiết**.
- Mỗi ứng dụng, mỗi vai trò nên có **user riêng**, để dễ kiểm soát và truy vết.

**Tạo user trước khi cấp quyền:**

```sql
CREATE USER 'shop_app'@'localhost' IDENTIFIED BY 'MatKhauManh#2026';
```

- `'shop_app'`: tên user.
- `'localhost'`: user chỉ được kết nối **từ máy này**. Dùng `'%'` cho phép từ mọi nơi (hạn chế dùng), hoặc IP cụ thể như `'10.0.0.5'`.
- Mật khẩu phải mạnh, và không đưa thẳng vào mã nguồn (dùng biến môi trường hoặc secret manager).

## GRANT - cấp quyền cho user

**Cú pháp:**

```sql
GRANT quyền_1, quyền_2 ON tên_database.tên_bảng TO 'user'@'host';
```

**Phạm vi áp dụng (`ON ...`):**

| Phạm vi | Ý nghĩa |
|---|---|
| `*.*` | Toàn bộ server (rất nguy hiểm) |
| `shop_db.*` | Mọi bảng trong database `shop_db` |
| `shop_db.products` | Chỉ một bảng |

**Ví dụ:** Cấp quyền cho ứng dụng bán hàng.

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON shop_db.*
TO 'shop_app'@'localhost';
```

> **Lưu ý:** Ở MySQL 8, `GRANT` không còn tự tạo user, phải `CREATE USER` trước. Quyền thay đổi có hiệu lực ngay, nhưng user đang kết nối có thể cần kết nối lại.

## Các loại quyền phổ biến trong MySQL

| Quyền | Cho phép | Nhóm |
|---|---|---|
| `SELECT` | Đọc dữ liệu | DML |
| `INSERT` | Thêm dòng | DML |
| `UPDATE` | Sửa dòng | DML |
| `DELETE` | Xóa dòng | DML |
| `CREATE` | Tạo database, bảng | DDL |
| `ALTER` | Sửa cấu trúc bảng | DDL |
| `DROP` | Xóa database, bảng | DDL |
| `INDEX` | Tạo, xóa index | DDL |
| `EXECUTE` | Chạy stored procedure, function | Khác |
| `ALL PRIVILEGES` | Tất cả quyền trong phạm vi chỉ định | Cần cẩn thận |
| `GRANT OPTION` | Cho phép user cấp lại quyền cho người khác | Rất hạn chế dùng |

**Ghi nhớ:** Ứng dụng khi chạy bình thường **chỉ cần nhóm DML** (`SELECT`, `INSERT`, `UPDATE`, `DELETE`). Quyền DDL (`CREATE`, `ALTER`, `DROP`) nên dành riêng cho user chạy migration.

## REVOKE - Thu hồi quyền truy cập

```sql
REVOKE quyền ON tên_database.tên_bảng FROM 'user'@'host';
```

**Ví dụ:**

```sql
-- Thu hồi quyền xóa
REVOKE DELETE ON shop_db.* FROM 'shop_app'@'localhost';

-- Thu hồi toàn bộ quyền
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'shop_app'@'localhost';
```

> **Lưu ý:** Phạm vi của `REVOKE` phải **khớp** với phạm vi đã `GRANT`. Cấp ở `shop_db.*` mà thu hồi ở `shop_db.orders` sẽ không có tác dụng.

## Kiểm tra quyền hiện tại của 1 User

```sql
-- Quyền của user chỉ định
SHOW GRANTS FOR 'shop_app'@'localhost';

-- Quyền của user đang đăng nhập
SHOW GRANTS;

-- Danh sách tất cả user
SELECT user, host FROM mysql.user;
```

**Kết quả mẫu:**

```
GRANT USAGE ON *.* TO `shop_app`@`localhost`
GRANT SELECT, INSERT, UPDATE, DELETE ON `shop_db`.* TO `shop_app`@`localhost`
```

`USAGE` nghĩa là "chỉ được kết nối, không có quyền nào khác" (dòng mặc định luôn xuất hiện).

## Column-Level Privileges - Phân quyền chi tiết đến từng cột

Giới hạn quyền xuống **từng cột**, hữu ích khi bảng có cột nhạy cảm (lương, giá vốn, số điện thoại).

```sql
GRANT SELECT (id, name, price), UPDATE (quantity)
ON shop_db.products
TO 'warehouse_staff'@'localhost';
```

**Kết quả với user này:**

```sql
SELECT id, name, price FROM products;        -- được
SELECT cost_price FROM products;             -- bị từ chối
UPDATE products SET quantity = 50 WHERE id = 1;   -- được
UPDATE products SET price = 1 WHERE id = 1;       -- bị từ chối
```

> **Lưu ý:** Phân quyền theo cột khó quản lý khi có nhiều cột. Thường dễ hơn nếu tạo **VIEW** chỉ chứa các cột được phép, rồi cấp quyền trên view đó.

```sql
CREATE VIEW v_products_public AS
SELECT id, name, price FROM products;

GRANT SELECT ON shop_db.v_products_public TO 'report_user'@'localhost';
```

## Usecase thực tế: Phân chia nhiều User theo vai trò trong hệ thống bán hàng

| User | Vai trò | Quyền cần thiết |
|---|---|---|
| `shop_app` | Ứng dụng Spring Boot chạy hằng ngày | DML trên `shop_db` |
| `shop_migration` | Chạy migration (Flyway, Liquibase) | DDL + DML trên `shop_db` |
| `report_user` | Báo cáo, thống kê | Chỉ `SELECT` |
| `backup_user` | Sao lưu | `SELECT`, `LOCK TABLES`, `SHOW VIEW` |
| `warehouse_staff` | Nhân viên kho | `SELECT` và `UPDATE` cột `quantity` |

```sql
-- 1. Ứng dụng: chỉ thao tác dữ liệu
CREATE USER 'shop_app'@'10.0.0.%' IDENTIFIED BY 'Mk_App#2026';
GRANT SELECT, INSERT, UPDATE, DELETE ON shop_db.* TO 'shop_app'@'10.0.0.%';

-- 2. Migration: được đổi cấu trúc, nhưng chỉ trong shop_db
CREATE USER 'shop_migration'@'localhost' IDENTIFIED BY 'Mk_Mig#2026';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, DROP, INDEX
ON shop_db.* TO 'shop_migration'@'localhost';

-- 3. Báo cáo: chỉ đọc
CREATE USER 'report_user'@'localhost' IDENTIFIED BY 'Mk_Rep#2026';
GRANT SELECT ON shop_db.* TO 'report_user'@'localhost';

-- 4. Sao lưu
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'Mk_Bak#2026';
GRANT SELECT, LOCK TABLES, SHOW VIEW ON shop_db.* TO 'backup_user'@'localhost';

-- 5. Nhân viên kho: giới hạn theo cột
CREATE USER 'warehouse_staff'@'localhost' IDENTIFIED BY 'Mk_Kho#2026';
GRANT SELECT (id, name, quantity), UPDATE (quantity)
ON shop_db.products TO 'warehouse_staff'@'localhost';
```

**Dùng ROLE để quản lý gọn hơn (MySQL 8.0):** gom quyền vào vai trò rồi gán cho nhiều user.

```sql
CREATE ROLE 'role_readonly';
GRANT SELECT ON shop_db.* TO 'role_readonly';

GRANT 'role_readonly' TO 'report_user'@'localhost';
SET DEFAULT ROLE 'role_readonly' TO 'report_user'@'localhost';
```

**Trong Spring Boot:** cấu hình user riêng cho ứng dụng, tách biến môi trường khỏi mã nguồn.

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/shop_db
    username: ${DB_USER}        # shop_app
    password: ${DB_PASSWORD}
```

## Xóa user không cần thiết

```sql
DROP USER 'warehouse_staff'@'localhost';

-- Không báo lỗi nếu user không tồn tại
DROP USER IF EXISTS 'warehouse_staff'@'localhost';
```

**Lưu ý:**
- `DROP USER` xóa luôn mọi quyền của user đó.
- User đang kết nối vẫn tiếp tục phiên hiện tại cho đến khi ngắt, nên cần kiểm tra bằng `SHOW PROCESSLIST` và ngắt kết nối nếu cần.
- Nên **rà soát định kỳ**: xóa tài khoản của nhân viên đã nghỉ, tài khoản test, tài khoản mặc định không còn dùng.

```sql
-- Kiểm tra tài khoản không có mật khẩu hoặc cho phép từ mọi nơi
SELECT user, host FROM mysql.user WHERE host = '%';
```

## Tổng kết

| Nội dung | Lệnh | Ghi nhớ |
|---|---|---|
| Tạo user | `CREATE USER` | Giới hạn `host`, mật khẩu mạnh |
| Cấp quyền | `GRANT ... ON ... TO ...` | Chỉ cấp tối thiểu cần thiết |
| Thu hồi | `REVOKE ... FROM ...` | Phạm vi phải khớp với lúc cấp |
| Kiểm tra | `SHOW GRANTS FOR ...` | Rà soát định kỳ |
| Theo cột | `GRANT SELECT (cột) ...` | Hoặc dùng `VIEW` cho dễ quản lý |
| Xóa user | `DROP USER` | Dọn tài khoản không còn dùng |

**Các lỗi hay gặp:**
- Dùng `root` cho ứng dụng.
- Cấp `ALL PRIVILEGES` hoặc `*.*` cho tiện.
- Dùng host `'%'` cho mọi user.
- Gộp user ứng dụng và user migration làm một, khiến ứng dụng có quyền `DROP TABLE`.

**Cách nhớ nhanh:** Phân quyền giống **chìa khóa tòa nhà**: nhân viên chỉ nhận chìa của phòng mình làm việc, không ai cầm chìa vạn năng (`root`) trừ quản trị viên, và khi nghỉ việc thì thu hồi chìa ngay.
