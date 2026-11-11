# Knowledge

## Lộ trình học MySQL cho việc xây dựng web bán hàng

### Giai đoạn 1: Nền tảng quan hệ & SQL cơ bản
- Khái niệm: bảng, hàng, cột, kiểu dữ liệu (`INT`, `VARCHAR`, `DECIMAL`, `DATETIME`, `ENUM`, `BOOLEAN`, `TEXT`,...)
- Khóa chính (PK), `AUTO_INCREMENT`
- Lệnh DDL: `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`
- Lệnh DML cơ bản: `INSERT`, `SELECT`, `UPDATE`, `DELETE`, `ORDER BY`, `LIMIT`

### Giai đoạn 2: Quan hệ giữa các bảng
- Khóa ngoại (FOREIGN KEY), ràng buộc referential integrity
- Các loại quan hệ: 1-1, 1-N, N-N (và bảng trung gian/junction table)
- ON DELETE CASCADE, SET NULL | RESTRICT
- JOIN: INNER JOIN, LEFT JOIN, RIGHT JOIN 

### Giai đoạn 3: Thiết kế & chuẩn hóa dữ liệu
- Khi nào nên phi chuẩn hóa
- Chuẩn hóa: 1NF, 2NF, 3NF - hiểu tại sao không nên nhét mọi thứ vào 1 bảng
- Phụ thuộc hàm cơ bản

### Giai đoạn 4: Truy vấn nâng cao
- GROUP BY, HAVING, các hàm tổng hợp (SUM, COUNT, AVG, MAX/MIN)
- Subquery
- UNION
- Window functions: ROW_NUMBER(), RANK() (MySQL 8+)
- CASE WHEN trong SELECT

### Giai đoạn 5: Transaction & tính toàn vẹn dữ liệu
- ACID là gì, tại sao cần transaction
- START TRANSACTION, COMMIT, ROLLBACK
- Isolation levels (ít nhất hiểu khái niệm, không cần sâu ngay)
- Xử lý race condition khi nhiểu người cùng mua 1 sản phẩm cuối cùng trong kho
  
### Giai đoạn 6: Index & tối ưu hiệu năng
- Index là gì, B-Tree index hoạt động ra sao
- Khi nào nên đánh index (cột hay JOIN, WHERE, ORDER BY) và khi nào không nên (cột hay update)
- Dùng EXPLAIN để đọc execution plan
- Composite index

### Giai đoạn 7: Bảo mật & quản trị
- Hash mật khẩu (Bcrypt), không bao giờ lưu plain text
- Phòng chống SQL Injection (dùng prepare statements)
- Phân quyền user trong MySQL (GRANT/REVOKE)
- Backup (`mysqldump`) và restore
