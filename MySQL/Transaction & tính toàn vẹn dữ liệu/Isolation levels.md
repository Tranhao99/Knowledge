# Isolation levels

## 1. Vấn đề: Tại sao cần nhiều mức cô lập khác nhau?

- Cô lập càng chặt thì dữ liệu càng an toàn, nhưng các transaction phải **chờ nhau nhiều hơn** nên hệ thống chạy chậm hơn. 
- Cô lập càng lỏng thì chạy nhanh hơn nhưng dễ đọc phải dữ liệu sai lệch.
- Mỗi nghiệp vụ cần mức an toàn khác nhau (báo cáo thống kê khác với chuyển tiền), nên SQL cung cấp 4 mức để bạn **cân bằng giữa độ chính xác và hiệu năng**.

## 2. Ba hiện tượng khi cô lập không đủ chặt

### a) Dirty read

- Transaction B đọc được dữ liệu mà Transaction A **chưa COMMIT**. Dữ liệu này là "bẩn": A có thể `ROLLBACK` sau đó, nên thứ B đã đọc chưa chắc là sự thật cuối cùng.

**Ví dụ:** Sản phẩm đang có giá 150.000.

| Bước | Transaction A | Transaction B |
|---|---|---|
| 1 | `UPDATE products SET price = 100000 WHERE id = 1;` (chưa commit) | |
| 2 | | `SELECT price FROM products WHERE id = 1;` → **100000** |
| 3 | `ROLLBACK;` (giá về lại 150000) | |

B đã đọc và có thể đã tính tiền theo giá 100.000, một giá **chưa từng tồn tại thật**.

### b) Non-Repeatable Read

- Trong cùng 1 transaction, đọc **cùng 1 dòng** 2 lần nhưng cho **2 kết quả khác nhau**, vì Transaction khác đã `COMMIT` thay đổi giữa 2 lần đọc.

**Ví dụ:**

| Bước | Transaction A | Transaction B |
|---|---|---|
| 1 | `SELECT price FROM products WHERE id = 1;` → **150000** | |
| 2 | | `UPDATE products SET price = 100000 WHERE id = 1;` `COMMIT;` |
| 3 | `SELECT price FROM products WHERE id = 1;` → **100000** | |

Trong cùng transaction A, cùng một câu query nhưng 2 lần ra 2 giá khác nhau.

### c) Phantom Read

- Tương tự Non-Repeatable Read nhưng xảy ra với **tập hợp nhiều dòng** thay vì 1 dòng cụ thể: **số lượng dòng** trả về giữa 2 lần chạy cùng 1 câu query trong cùng 1 transaction bị thay đổi, vì Transaction khác đã `INSERT` hoặc `DELETE` dòng và `COMMIT`.

**Ví dụ:**

| Bước | Transaction A | Transaction B |
|---|---|---|
| 1 | `SELECT COUNT(*) FROM orders WHERE status = 'PENDING';` → **5** | |
| 2 | | `INSERT INTO orders (user_id, status) VALUES (2, 'PENDING');` `COMMIT;` |
| 3 | `SELECT COUNT(*) FROM orders WHERE status = 'PENDING';` → **6** | |

Dòng thứ 6 xuất hiện như một "bóng ma" giữa hai lần đọc.

**Phân biệt nhanh:**

| Hiện tượng | Dữ liệu bị đọc sai do | Xảy ra với |
|---|---|---|
| Dirty read | Thay đổi **chưa commit** | Một dòng |
| Non-Repeatable Read | Thay đổi (`UPDATE`) **đã commit** | Một dòng |
| Phantom Read | Dòng mới / bị xóa (`INSERT`/`DELETE`) **đã commit** | Tập nhiều dòng |

## 3. Bốn mức độ Isolation levels - từ lỏng nhất đến chặt nhất

| Mức cô lập | Dirty read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | Có thể xảy ra | Có thể xảy ra | Có thể xảy ra |
| READ COMMITTED | Không | Có thể xảy ra | Có thể xảy ra |
| REPEATABLE READ | Không | Không | Có thể xảy ra (theo chuẩn SQL) |
| SERIALIZABLE | Không | Không | Không |

> **Lưu ý với MySQL InnoDB:** Ở mức `REPEATABLE READ`, InnoDB dùng snapshot (MVCC) và khóa next-key nên **phần lớn tình huống phantom read cũng không xảy ra**. Bảng trên là theo chuẩn SQL, thực tế InnoDB chặt hơn một chút.

### a) READ UNCOMMITTED

Lỏng nhất. Transaction đọc được cả dữ liệu **chưa commit** của transaction khác. Nhanh nhất nhưng **không an toàn**, gần như không dùng trong thực tế.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

### b) READ COMMITTED

Chỉ đọc dữ liệu **đã commit**. Mỗi câu `SELECT` thấy dữ liệu mới nhất đã commit tại thời điểm chạy. Đây là mặc định của PostgreSQL, Oracle, SQL Server. Phù hợp khi chấp nhận dữ liệu thay đổi giữa các lần đọc.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### c) REPEATABLE READ

**Mặc định của MySQL InnoDB.** Transaction đọc từ một "ảnh chụp" dữ liệu tại lần đọc đầu tiên, nên các lần đọc sau trong cùng transaction luôn **cho kết quả giống nhau**, dù transaction khác đã commit thay đổi.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### d) SERIALIZABLE

Chặt nhất. Các transaction được xử lý **như thể chạy lần lượt từng cái một**: câu `SELECT` thường tự động bị khóa đọc, nên transaction khác muốn ghi vào dữ liệu đó phải chờ. An toàn tuyệt đối nhưng **chậm nhất** và dễ xảy ra deadlock hoặc chờ khóa lâu.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## 4. Cách thiết lập và kiểm tra

**Trong MySQL:**

```sql
-- Kiểm tra mức hiện tại
SELECT @@transaction_isolation;

-- Đặt cho phiên hiện tại
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Đặt chỉ cho transaction kế tiếp
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
```

**Trong Spring Boot:**

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processOrder(Long orderId) {
    // ...
}
```

Các giá trị: `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`. Nếu không khai báo, Spring dùng `Isolation.DEFAULT` (mặc định của database).

## 5. Chọn mức nào trong ứng dụng bán hàng?

- **Phần lớn nghiệp vụ:** giữ mặc định `REPEATABLE READ` của MySQL, hoặc dùng `READ COMMITTED` nếu muốn ít khóa hơn.
- **Báo cáo, thống kê cần số liệu nhất quán trong suốt quá trình:** `REPEATABLE READ`.
- **Trừ tồn kho, chuyển tiền (cần chính xác tuyệt đối):** không nên chỉ dựa vào isolation level. Dùng thêm **khóa dòng** (`SELECT ... FOR UPDATE`) hoặc **optimistic locking** (`@Version` trong JPA).
- **Tránh** `READ UNCOMMITTED`, và chỉ dùng `SERIALIZABLE` cho đoạn nhỏ thật sự cần vì rất chậm.

## Tổng kết

| Mức | Cách nhớ |
|---|---|
| READ UNCOMMITTED | "Đọc cả dữ liệu người khác chưa lưu" |
| READ COMMITTED | "Chỉ đọc dữ liệu đã lưu, mỗi lần đọc thấy bản mới nhất" |
| REPEATABLE READ | "Chụp ảnh một lần, đọc lại vẫn thấy y như cũ" |
| SERIALIZABLE | "Xếp hàng lần lượt, không ai chen ngang" |

**Cách nhớ nhanh:** Mức càng cao thì càng ít hiện tượng lỗi nhưng càng chậm. Ba hiện tượng đi theo thứ tự: **Dirty read** (đọc chưa commit) → **Non-Repeatable Read** (một dòng đổi giá trị) → **Phantom Read** (số dòng thay đổi).

Khi bạn sẵn sàng, mình có thể làm tiếp **Locking** (khóa dòng, `SELECT ... FOR UPDATE`, optimistic/pessimistic locking, deadlock).
