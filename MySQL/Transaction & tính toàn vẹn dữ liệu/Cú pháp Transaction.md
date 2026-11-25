# Cú pháp Transaction

## 1. AUTOCOMMIT - Chế độ mặc định

MySQL mặc định **bật `AUTOCOMMIT`**: mỗi câu lệnh SQL tự động là **một transaction riêng** và được lưu ngay sau khi chạy xong. Ví dụ `UPDATE` xong là dữ liệu đã được lưu vĩnh viễn, không thể `ROLLBACK`. Vì vậy, muốn gộp nhiều câu lệnh thành một transaction, ta phải **chủ động bắt đầu transaction** bằng `START TRANSACTION`. Khi đó `AUTOCOMMIT` được tạm tắt cho đến khi bạn `COMMIT` hoặc `ROLLBACK`.

## 2. Cú pháp cơ bản

```sql
START TRANSACTION;   -- hoặc BEGIN

-- các câu lệnh SQL (INSERT, UPDATE, DELETE...)

COMMIT;              -- lưu tất cả
-- hoặc
ROLLBACK;            -- hủy tất cả
```

**Ví dụ:** Đặt hàng.

```sql
START TRANSACTION;

INSERT INTO orders (user_id, total_amount) VALUES (1, 300000);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 5, 2);
UPDATE products SET quantity = quantity - 2 WHERE id = 5;

COMMIT;
```

> **Lưu ý:** Nên dùng `START TRANSACTION` thay vì `BEGIN`, vì `BEGIN` dễ nhầm với khối `BEGIN ... END` trong stored procedure.

## 3. COMMIT

`COMMIT` **xác nhận và lưu vĩnh viễn** mọi thay đổi trong transaction, đồng thời kết thúc transaction và giải phóng các khóa (lock) đang giữ. Sau `COMMIT`, các transaction khác mới nhìn thấy thay đổi, và không thể hoàn tác nữa.

```sql
START TRANSACTION;
UPDATE products SET price = 160000 WHERE id = 1;
COMMIT;   -- từ đây giá mới được lưu vĩnh viễn
```

## 4. ROLLBACK

`ROLLBACK` **hủy toàn bộ thay đổi** từ lúc `START TRANSACTION`, đưa dữ liệu về trạng thái ban đầu và kết thúc transaction.

```sql
START TRANSACTION;
DELETE FROM products WHERE category_id = 2;   -- lỡ tay xóa nhầm
ROLLBACK;                                     -- dữ liệu trở lại như cũ
```

**Những điều cần nhớ:**
- Khi một câu lệnh bị lỗi (ví dụ vi phạm khóa ngoại), MySQL thường **chỉ hủy riêng câu đó**, transaction **vẫn mở**. Bạn phải tự quyết định `COMMIT` hay `ROLLBACK`. Riêng lỗi **deadlock** thì MySQL tự rollback cả transaction.
- Nếu **mất kết nối** khi chưa `COMMIT`, MySQL tự động `ROLLBACK`.

## 5. Kiểm soát bằng code thực tế

Trong Spring Boot (JPA + Hibernate), ta không viết `START TRANSACTION` thủ công mà dùng annotation **`@Transactional`**. Spring tự động:
- Bắt đầu transaction khi vào method.
- `COMMIT` khi method chạy xong bình thường.
- `ROLLBACK` khi method ném ra exception.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    @Transactional(rollbackFor = Exception.class)
    public Order placeOrder(Long userId, Long productId, int quantity) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new IllegalArgumentException("Sản phẩm không tồn tại"));

        if (product.getQuantity() < quantity) {
            throw new IllegalStateException("Không đủ hàng");   // -> ROLLBACK
        }

        product.setQuantity(product.getQuantity() - quantity);  // Hibernate tự UPDATE khi commit

        Order order = new Order();
        order.setUserId(userId);
        order.setTotalAmount(product.getPrice().multiply(BigDecimal.valueOf(quantity)));

        return orderRepository.save(order);                     // hết method -> COMMIT
    }
}
```

**Với truy vấn chỉ đọc**, dùng `readOnly = true` để tối ưu:

```java
@Transactional(readOnly = true)
public List<Order> getOrders(Long userId) {
    return orderRepository.findByUserId(userId);
}
```

**Các lỗi hay gặp với `@Transactional`:**
- **Mặc định chỉ rollback với `RuntimeException` và `Error`.** Với checked exception (như `IOException`, `Exception`) thì vẫn **commit**. Cần thêm `rollbackFor = Exception.class`.
- **Nuốt exception bằng `try-catch`** mà không ném lại thì Spring tưởng method thành công và sẽ `COMMIT`.
- **Gọi method `@Transactional` từ method khác trong cùng class** (self-invocation) thì annotation **không có tác dụng**, vì không đi qua proxy của Spring. Nên tách sang bean khác.
- **Chỉ có hiệu lực với method `public`.**
- Nên đặt `@Transactional` ở tầng **Service**, không đặt ở Controller hay Repository.

## 6. SAVEPOINT - Rollback từng phần, không cần hủy toàn bộ Transaction

`SAVEPOINT` đánh dấu **một điểm giữa transaction**, để sau này có thể quay về đúng điểm đó mà **không hủy** những gì làm trước nó.

```sql
SAVEPOINT tên_điểm;                  -- tạo điểm đánh dấu
ROLLBACK TO SAVEPOINT tên_điểm;      -- quay về điểm đó (transaction vẫn mở)
RELEASE SAVEPOINT tên_điểm;          -- xóa điểm đánh dấu
```

**Ví dụ:** Tạo đơn hàng, thử thêm quà tặng, nếu hết quà thì bỏ quà nhưng vẫn giữ đơn.

```sql
START TRANSACTION;

INSERT INTO orders (user_id, total_amount) VALUES (1, 300000);
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (101, 99, 1);  -- thử thêm quà
-- quà hết hàng, chỉ hủy phần này
ROLLBACK TO SAVEPOINT after_order;

COMMIT;   -- đơn hàng vẫn được lưu, phần quà thì không
```

**Lưu ý:**
- `ROLLBACK TO SAVEPOINT` **không kết thúc transaction**, vẫn phải `COMMIT` hoặc `ROLLBACK` ở cuối.
- Các savepoint tạo **sau** điểm được quay về sẽ bị xóa.
- Trong Spring có `Propagation.NESTED` tương ứng với savepoint, nhưng ít dùng trong thực tế.

## 7. Lưu ý về DDL trong transaction - MySQL có giới hạn quan trọng

Trong MySQL, các lệnh **DDL** (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`) gây ra **implicit commit**: MySQL **tự động `COMMIT`** transaction đang mở **trước và sau** khi chạy lệnh DDL. Hệ quả là **không thể rollback DDL**, và phần DML bạn làm trước đó cũng đã bị lưu mà không hề báo.

```sql
START TRANSACTION;

UPDATE products SET price = 160000 WHERE id = 1;   -- (1)
ALTER TABLE products ADD COLUMN note TEXT;         -- DDL: tự COMMIT cả (1)

ROLLBACK;   -- vô tác dụng, giá đã được lưu rồi
```

**Điều cần nhớ:**
- Từ MySQL 8.0, mỗi lệnh DDL riêng lẻ là **atomic** (chạy trọn vẹn hoặc không chạy), nhưng vẫn **kèm commit ngầm**.
- Một số lệnh khác cũng gây commit ngầm, như `LOCK TABLES` hay một `START TRANSACTION` mới khi đang có transaction cũ.
- **Cách làm đúng:** không trộn DDL với DML trong cùng transaction. Thay đổi cấu trúc bảng nên chạy **riêng**, quản lý bằng công cụ migration như **Flyway** hoặc **Liquibase**.

## 8. Kiểm tra trạng thái AUTOCOMMIT hiện tại

```sql
SELECT @@autocommit;              -- 1 là bật, 0 là tắt

SHOW VARIABLES LIKE 'autocommit'; -- hiển thị ON / OFF
```

**Bật/tắt cho phiên làm việc hiện tại:**

```sql
SET autocommit = 0;   -- tắt: mọi lệnh nằm trong transaction đến khi COMMIT/ROLLBACK
SET autocommit = 1;   -- bật lại (mặc định)
```

**Lưu ý:**
- `SET autocommit` chỉ có tác dụng trong **phiên (session) hiện tại**.
- Nếu tắt `AUTOCOMMIT` mà **quên `COMMIT`**, transaction mở mãi, giữ khóa và làm các phiên khác bị chờ hoặc treo. Hãy cẩn thận khi dùng công cụ như MySQL Workbench hoặc DBeaver có chế độ tắt autocommit.
- Với Spring Boot, bạn không cần tự đổi. Connection pool (HikariCP) mặc định để `autoCommit = true`, và Spring tự tắt tạm thời khi vào method `@Transactional`.

## Tổng kết

| Lệnh | Tác dụng |
|---|---|
| `START TRANSACTION` | Bắt đầu transaction, tạm tắt autocommit |
| `COMMIT` | Lưu vĩnh viễn, kết thúc transaction |
| `ROLLBACK` | Hủy toàn bộ thay đổi, kết thúc transaction |
| `SAVEPOINT` / `ROLLBACK TO` | Đánh dấu điểm và hủy một phần |
| `SELECT @@autocommit` | Kiểm tra chế độ autocommit |
| `@Transactional` (Spring) | Tự động `START` / `COMMIT` / `ROLLBACK` theo method |

**Cách nhớ nhanh:** Mặc định mỗi lệnh tự lưu. Muốn gộp nhiều lệnh thì `START TRANSACTION`, xong thì `COMMIT` để lưu hoặc `ROLLBACK` để hủy. Trong Spring Boot, chỉ cần `@Transactional` và nhớ `rollbackFor = Exception.class` nếu có checked exception. Tránh trộn DDL vào transaction vì nó sẽ tự commit.
