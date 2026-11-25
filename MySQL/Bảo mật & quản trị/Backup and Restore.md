# Backup & Restore

> **Backup (sao lưu)** là tạo bản sao dữ liệu để khôi phục khi gặp sự cố: xóa nhầm, hỏng đĩa, bị tấn công, lỗi khi nâng cấp. **Restore (khôi phục)** là đưa dữ liệu từ bản sao lưu trở lại database. Backup chưa từng được thử restore thì **chưa thể coi là backup**.

## 1. Các loại backup

| Tiêu chí | Loại | Ý nghĩa |
|---|---|---|
| Hình thức | **Logical** | Xuất ra file SQL (câu lệnh tạo bảng và INSERT). Dễ đọc, dễ chuyển sang server khác. Dùng `mysqldump` |
| | **Physical** | Sao chép trực tiếp file dữ liệu. Nhanh với database lớn, khó chuyển phiên bản. Dùng Percona XtraBackup |
| Phạm vi | **Full** | Sao lưu toàn bộ dữ liệu |
| | **Incremental** | Chỉ sao lưu phần thay đổi từ lần backup trước (qua binary log) |

**Chọn nhanh:** Database dưới vài chục GB thì `mysqldump` là đủ. Database lớn hơn thì cân nhắc backup physical.

## 2. Backup bằng `mysqldump`

Chạy ở **terminal**, không phải trong MySQL client.

**Backup một database:**

```bash
mysqldump -u backup_user -p \
  --single-transaction \
  --routines --triggers --events \
  shop_db > shop_db_backup.sql
```

| Tùy chọn | Ý nghĩa |
|---|---|
| `--single-transaction` | Tạo bản backup **nhất quán** mà không khóa bảng (với InnoDB). Gần như luôn nên dùng |
| `--routines` | Kèm stored procedure, function |
| `--triggers` | Kèm trigger (mặc định đã bật) |
| `--events` | Kèm event |

**Các cách backup khác:**

```bash
# Nhiều database
mysqldump -u root -p --databases shop_db blog_db > multi.sql

# Toàn bộ server
mysqldump -u root -p --all-databases --single-transaction > all.sql

# Chỉ một số bảng
mysqldump -u root -p shop_db products orders > two_tables.sql

# Chỉ cấu trúc, không dữ liệu
mysqldump -u root -p --no-data shop_db > schema_only.sql
```

**Nén ngay khi backup, kèm ngày tháng trong tên file:**

```bash
mysqldump -u backup_user -p --single-transaction shop_db \
  | gzip > shop_db_$(date +%F).sql.gz
```

> **Lưu ý:** Không gõ mật khẩu thẳng sau `-p` trên dòng lệnh (bị lộ trong lịch sử lệnh). Để trống để MySQL hỏi, hoặc dùng file cấu hình `~/.my.cnf` (đặt quyền `600`).

## 3. Restore (khôi phục)

**Restore từ file `.sql`:**

```bash
# Cách 1: dùng redirect
mysql -u root -p shop_db < shop_db_backup.sql

# Cách 2: từ trong MySQL client
mysql> SOURCE /path/to/shop_db_backup.sql;
```

**Restore từ file nén:**

```bash
gunzip < shop_db_2026-11-25.sql.gz | mysql -u root -p shop_db
```

**Nếu database chưa tồn tại**, tạo trước:

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS shop_db CHARACTER SET utf8mb4;"
mysql -u root -p shop_db < shop_db_backup.sql
```

**Restore chỉ một bảng:** `mysqldump` mặc định có lệnh `DROP TABLE IF EXISTS` trước khi tạo lại bảng, nên restore sẽ **ghi đè** dữ liệu hiện tại. Cần cẩn thận, tốt nhất thử trên database tạm trước.

```bash
mysql -u root -p -e "CREATE DATABASE shop_db_test;"
mysql -u root -p shop_db_test < shop_db_backup.sql
```

## 4. Point-in-Time Recovery (khôi phục về một thời điểm)

Backup full chỉ đưa dữ liệu về **thời điểm sao lưu**. Nếu muốn khôi phục đến thời điểm cụ thể (ví dụ ngay trước khi ai đó chạy `DELETE` nhầm lúc 14:30), cần kết hợp **binary log (binlog)**: file ghi lại mọi thay đổi dữ liệu.

**Kiểm tra binlog đã bật chưa** (MySQL 8 mặc định bật):

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW BINARY LOGS;
```

**Quy trình:**
1. Restore bản backup full gần nhất.
2. Dùng `mysqlbinlog` phát lại các thay đổi từ sau backup đến **ngay trước thời điểm sự cố**.

```bash
mysqlbinlog --start-datetime="2026-11-25 02:00:00" \
            --stop-datetime="2026-11-25 14:29:59" \
            --database=shop_db \
            /var/lib/mysql/binlog.000012 | mysql -u root -p
```

> Muốn dùng được cách này, **binlog phải được giữ lại** (không xóa sớm) và nên sao lưu cả binlog sang nơi khác.

## 5. Tự động hóa backup

**Script backup đơn giản** (`backup.sh`):

```bash
#!/bin/bash
BACKUP_DIR=/var/backups/mysql
DATE=$(date +%F_%H%M)
mkdir -p "$BACKUP_DIR"

mysqldump --defaults-extra-file=/etc/mysql/backup.cnf \
  --single-transaction --routines --triggers --events \
  shop_db | gzip > "$BACKUP_DIR/shop_db_$DATE.sql.gz"

# Xóa backup cũ hơn 14 ngày
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +14 -delete
```

**Lập lịch chạy bằng `cron`** (mỗi ngày lúc 2 giờ sáng):

```bash
0 2 * * * /opt/scripts/backup.sh >> /var/log/mysql_backup.log 2>&1
```

**User riêng cho backup** (quyền tối thiểu, không dùng `root`):

```sql
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'Mk_Bak#2026';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON shop_db.* TO 'backup_user'@'localhost';
```

## 6. Nguyên tắc backup an toàn

- **Quy tắc 3-2-1:** có **3** bản sao dữ liệu, trên **2** loại lưu trữ khác nhau, trong đó **1** bản nằm ở **nơi khác** (cloud, server khác). Để backup cùng ổ đĩa với database thì ổ hỏng là mất tất cả.
- **Thử restore định kỳ:** thực sự khôi phục lên server thử nghiệm để chắc backup dùng được.
- **Mã hóa và phân quyền** file backup, vì nó chứa toàn bộ dữ liệu (kể cả thông tin nhạy cảm).
- **Giữ lịch sử hợp lý:** ví dụ giữ hằng ngày trong 14 ngày, hằng tuần trong 2 tháng, hằng tháng trong 1 năm.
- **Theo dõi và cảnh báo:** có thông báo khi backup thất bại hoặc dung lượng file bất thường (quá nhỏ).
- **Backup trước khi làm việc rủi ro:** nâng cấp phiên bản, chạy migration lớn, xóa dữ liệu hàng loạt.

## 7. Tần suất backup: RPO và RTO

| Khái niệm | Ý nghĩa | Ví dụ |
|---|---|---|
| **RPO** (Recovery Point Objective) | Chấp nhận **mất tối đa bao nhiêu dữ liệu** | RPO 1 giờ: backup (hoặc binlog) ít nhất mỗi giờ |
| **RTO** (Recovery Time Objective) | Chấp nhận **mất bao lâu** để hệ thống hoạt động lại | RTO 30 phút: quy trình restore phải xong trong 30 phút |

Với ứng dụng bán hàng, thường kết hợp: **full backup mỗi đêm + giữ binlog liên tục** để khôi phục về thời điểm bất kỳ trong ngày.

## Tổng kết

| Công việc | Lệnh |
|---|---|
| Backup một database | `mysqldump --single-transaction shop_db > file.sql` |
| Backup nén | `mysqldump ... \| gzip > file.sql.gz` |
| Restore | `mysql -u root -p shop_db < file.sql` |
| Khôi phục về thời điểm | Restore full + `mysqlbinlog --stop-datetime=...` |
| Tự động hóa | Script + `cron` |

**Các lỗi hay gặp:**
- Quên `--single-transaction` nên bản backup không nhất quán hoặc khóa bảng.
- Backup cùng máy chủ với database, không có bản ngoài nơi khác.
- Không bao giờ thử restore.
- Restore đè lên database đang chạy mà chưa kiểm tra.
- Để mật khẩu lộ trong script hoặc lịch sử lệnh.

**Cách nhớ nhanh:** Backup là **bảo hiểm**, restore là **lúc đòi bảo hiểm**. Backup tốt là backup được lưu ở nhiều nơi, được tự động hóa, và đã được **thử khôi phục thành công**.
