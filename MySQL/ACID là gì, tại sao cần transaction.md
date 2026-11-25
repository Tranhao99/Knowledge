# ACID là gì, tại sao cần transaction

## 1. Điều gì xảy ra khi 1 nghiệp vụ cần nhiều câu lệnh SQL?

Một nghiệp vụ thực tế thường gồm **nhiều câu lệnh SQL** phải chạy liên tiếp. Nếu chạy được vài câu rồi bị lỗi (mất mạng, server sập, vi phạm ràng buộc), dữ liệu sẽ rơi vào trạng thái **dở dang**: tiền đã trừ nhưng đơn chưa tạo, đơn đã tạo nhưng kho chưa trừ. Hai người cùng thao tác một lúc cũng có thể làm dữ liệu sai lệch.

**Ví dụ:** Đặt hàng gồm 3 bước.

```sql
INSERT INTO orders (user_id, total_amount) VALUES (1, 300000);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (101, 5, 2);
UPDATE products SET quantity = quantity - 2 WHERE id = 5;
```

Nếu lỗi xảy ra sau bước 2, đơn hàng đã có nhưng **kho chưa bị trừ**, dẫn đến bán quá số lượng thực tế.

## 2. Transaction là gì

**Transaction (giao dịch)** là một nhóm câu lệnh SQL được gộp lại thành **một đơn vị công việc không thể chia nhỏ**: hoặc **tất cả cùng thành công** (`COMMIT`), hoặc **tất cả bị hủy** như chưa từng xảy ra (`ROLLBACK`). Không có trường hợp thành công một nửa.

```sql
START TRANSACTION;

INSERT INTO orders (user_id, total_amount) VALUES (1, 300000);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (101, 5, 2);
UPDATE products SET quantity = quantity - 2 WHERE id = 5;

COMMIT;      -- lưu tất cả
-- hoặc ROLLBACK;  -- hủy tất cả
```

> **Lưu ý:** Cần dùng bảng engine **InnoDB** (mặc định của MySQL) mới hỗ trợ transaction. Trong Spring Boot, dùng `@Transactional` để quản lý.

## 3. ACID - 4 tính chất một Transaction phải đảm bảo

**ACID** là viết tắt của 4 tính chất giúp transaction đáng tin cậy: **A**tomicity, **C**onsistency, **I**solation, **D**urability.

### A - Atomicity (tính nguyên tử)

**"Tất cả hoặc không gì cả."** Mọi câu lệnh trong transaction được coi là một khối duy nhất. Nếu bất kỳ câu nào lỗi, toàn bộ những gì đã làm trước đó đều bị **hoàn tác (rollback)**, dữ liệu quay về trạng thái ban đầu. Nhờ vậy không bao giờ có chuyện "làm dở".

**Ví dụ:** Chuyển 100.000 từ ví A sang ví B. Trừ tiền A thành công nhưng cộng tiền B lỗi thì **cả hai bước đều bị hủy**, tiền không bị mất.

```sql
START TRANSACTION;
UPDATE wallets SET balance = balance - 100000 WHERE user_id = 1;
UPDATE wallets SET balance = balance + 100000 WHERE user_id = 2;
COMMIT;
```

### C - Consistency (tính nhất quán)

Transaction đưa database từ **một trạng thái hợp lệ sang một trạng thái hợp lệ khác**, luôn tuân thủ mọi quy tắc đã định nghĩa: khóa chính, khóa ngoại, `NOT NULL`, `UNIQUE`, `CHECK`. Nếu transaction làm vi phạm quy tắc thì bị từ chối và rollback. Lưu ý các quy tắc nghiệp vụ như "tổng tiền hai ví không đổi" vẫn do **lập trình viên** bảo đảm trong logic.

**Ví dụ:** Số lượng tồn kho không được âm.

```sql
ALTER TABLE products
ADD CONSTRAINT chk_quantity CHECK (quantity >= 0);

-- Lệnh này bị từ chối nếu kho chỉ còn 1 sản phẩm
UPDATE products SET quantity = quantity - 2 WHERE id = 5;
```

### I - Isolation (tính cô lập)

Nhiều transaction chạy **cùng lúc** không được làm ảnh hưởng sai lệch lẫn nhau. Mỗi transaction hoạt động như thể nó là người duy nhất đang dùng database, và không thấy những thay đổi **chưa commit** của transaction khác. Mức độ cô lập có thể điều chỉnh (isolation level), mức càng cao càng an toàn nhưng càng chậm.

**Ví dụ:** Kho còn 1 sản phẩm, 2 khách đặt cùng lúc. Nhờ cô lập (kết hợp khóa dòng), chỉ một người mua được, người còn lại phải chờ rồi thấy kho đã hết.

```sql
START TRANSACTION;
SELECT quantity FROM products WHERE id = 5 FOR UPDATE;  -- khóa dòng này
UPDATE products SET quantity = quantity - 1 WHERE id = 5;
COMMIT;
```

### D - Durability (tính bền vững)

Khi transaction đã **`COMMIT` thành công**, dữ liệu được ghi xuống đĩa và **được giữ lại vĩnh viễn**, kể cả khi server mất điện hay sập ngay sau đó. InnoDB đạt được điều này nhờ **redo log**: ghi lại thay đổi trước, khi khởi động lại sẽ khôi phục những gì đã commit. Ngược lại, transaction chưa commit mà bị sập thì được hoàn tác.

**Ví dụ:** Sau khi khách thanh toán và `COMMIT`, đơn hàng vẫn còn nguyên dù server khởi động lại ngay lúc đó.

## 4. Tổng kết

| Tính chất | Ý nghĩa ngắn gọn | Bảo vệ khỏi lỗi gì |
|---|---|---|
| **A**tomicity | Tất cả hoặc không gì cả | Dữ liệu làm dở dang |
| **C**onsistency | Luôn tuân thủ quy tắc dữ liệu | Dữ liệu sai quy tắc, vô lý |
| **I**solation | Các transaction không ảnh hưởng lẫn nhau | Xung đột khi nhiều người cùng thao tác |
| **D**urability | Đã commit thì không mất | Mất dữ liệu khi server sập |

**Các lệnh cần nhớ:**

| Lệnh | Tác dụng |
|---|---|
| `START TRANSACTION` | Bắt đầu transaction |
| `COMMIT` | Xác nhận, lưu vĩnh viễn |
| `ROLLBACK` | Hủy toàn bộ thay đổi từ lúc bắt đầu |

**Cách nhớ nhanh:** A là "không làm dở", C là "không phạm luật", I là "không dẫm chân nhau", D là "đã lưu là không mất".

Khi bạn sẵn sàng, mình có thể làm tiếp **Isolation level** (các mức cô lập và các hiện tượng như dirty read, non-repeatable read, phantom read) hoặc **Locking** cho phần Transaction.
