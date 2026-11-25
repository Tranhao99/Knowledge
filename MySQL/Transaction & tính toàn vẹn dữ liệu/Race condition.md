# Race Condition

> **Race condition (tình huống đua)** xảy ra khi nhiều transaction cùng đọc và sửa một dữ liệu, kết quả cuối cùng phụ thuộc vào **thứ tự chạy ngẫu nhiên** của chúng và có thể bị sai.

**Bài toán:** Xử lý race condition khi 2 khách hàng cùng mua 1 sản phẩm duy nhất còn trong kho.

## 1. Tái hiện chính xác vấn đề

Kho còn **1 sản phẩm**. Cả hai khách cùng đọc số lượng, cùng thấy "còn hàng", rồi cùng trừ kho và tạo đơn. Kết quả: **2 đơn hàng cho 1 sản phẩm** (bán vượt kho). Lỗi nằm ở chỗ "kiểm tra" và "cập nhật" là hai bước tách rời, người thứ hai chen vào giữa.

| Bước | Khách A | Khách B | quantity |
|---|---|---|---|
| 1 | `SELECT quantity` → 1 | | 1 |
| 2 | | `SELECT quantity` → 1 | 1 |
| 3 | 1 > 0, đủ hàng, tạo đơn | 1 > 0, đủ hàng, tạo đơn | 1 |
| 4 | `UPDATE SET quantity = 0` | `UPDATE SET quantity = 0` | 0 |

Code gây lỗi trong Spring Boot (đọc rồi ghi ở tầng ứng dụng):

```java
@Transactional
public void buy(Long productId) {
    Product p = productRepository.findById(productId).orElseThrow();
    if (p.getQuantity() < 1) throw new IllegalStateException("Hết hàng");
    p.setQuantity(p.getQuantity() - 1);   // cả A và B cùng tính ra 0
    orderRepository.save(new Order(productId));
}
```

## Các giải pháp

### Giải pháp 1: Pessimistic Locking (khóa bi quan) bằng `SELECT ... FOR UPDATE`

**Ý tưởng:** "Cứ cho rằng xung đột sẽ xảy ra", nên **khóa dòng dữ liệu ngay khi đọc**. Ai đến sau phải chờ.

**Cơ chế hoạt động:**
1. A chạy `SELECT ... FOR UPDATE`, InnoDB đặt **khóa ghi (exclusive lock)** lên dòng sản phẩm.
2. B chạy cùng câu lệnh, bị **chặn lại** cho đến khi A `COMMIT` hoặc `ROLLBACK`.
3. Khi B được chạy tiếp, nó đọc được giá trị **mới nhất** (quantity = 0) nên từ chối đơn hàng.

```sql
START TRANSACTION;

SELECT quantity FROM products WHERE id = 5 FOR UPDATE;   -- khóa dòng
-- kiểm tra quantity >= 1 trong ứng dụng
UPDATE products SET quantity = quantity - 1 WHERE id = 5;
INSERT INTO orders (user_id, product_id) VALUES (1, 5);

COMMIT;   -- nhả khóa
```

**Áp dụng trong Spring Boot (JPA):**

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)               // sinh ra SELECT ... FOR UPDATE
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;

    @Transactional
    public Order buy(Long userId, Long productId) {
        Product p = productRepository.findByIdForUpdate(productId)   // khóa dòng
                .orElseThrow(() -> new IllegalArgumentException("Sản phẩm không tồn tại"));

        if (p.getQuantity() < 1) {
            throw new IllegalStateException("Hết hàng");
        }

        p.setQuantity(p.getQuantity() - 1);
        return orderRepository.save(new Order(userId, productId));
    }   // COMMIT ở đây, khóa được nhả
}
```

**Ưu điểm:** Đơn giản về tư duy, an toàn tuyệt đối, phù hợp khi xung đột xảy ra thường xuyên.
**Nhược điểm:** Các transaction phải xếp hàng chờ nên **giảm thông lượng**, có nguy cơ **deadlock**.
**Lưu ý:** Giữ transaction **càng ngắn càng tốt** (không gọi API ngoài, không xử lý nặng khi đang giữ khóa) và đặt `lock.timeout` để không chờ vô hạn.

### Giải pháp 2: Atomic UPDATE với điều kiện

**Ý tưởng:** Gộp "kiểm tra" và "cập nhật" thành **một câu lệnh duy nhất**, để database tự bảo đảm tính nguyên tử. Không cần đọc trước.

```sql
UPDATE products
SET quantity = quantity - 1
WHERE id = 5
  AND quantity >= 1;
```

**Cơ chế:**
- InnoDB tự khóa dòng trong thời gian rất ngắn khi chạy câu `UPDATE`.
- Điều kiện `quantity >= 1` được kiểm tra **ngay lúc ghi**, trên dữ liệu mới nhất.
- Kiểm tra **số dòng bị ảnh hưởng (affected rows)**: `1` là mua thành công, `0` là đã hết hàng.

Với 2 khách cùng lúc: khách đến trước trừ thành công (quantity về 0), khách đến sau thấy `quantity >= 1` sai nên không dòng nào được cập nhật.

**Áp dụng trong Spring Boot:**

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Modifying(clearAutomatically = true)
    @Query("""
        UPDATE Product p
        SET p.quantity = p.quantity - :qty
        WHERE p.id = :id AND p.quantity >= :qty
        """)
    int decreaseStock(@Param("id") Long id, @Param("qty") int qty);
}
```

```java
@Transactional
public Order buy(Long userId, Long productId) {
    int updated = productRepository.decreaseStock(productId, 1);

    if (updated == 0) {
        throw new IllegalStateException("Hết hàng");   // ROLLBACK
    }

    return orderRepository.save(new Order(userId, productId));
}
```

**Ưu điểm:** Đơn giản, **nhanh nhất**, giữ khóa rất ngắn, không cần `SELECT` trước.
**Nhược điểm:** Chỉ hợp khi logic kiểm tra diễn đạt được trong một câu `UPDATE`. Nếu cần đọc dữ liệu để tính toán phức tạp rồi mới quyết định thì dùng giải pháp khác.

> **Nên chọn:** Với bài toán trừ tồn kho, đây thường là **lựa chọn đầu tiên**.

### Giải pháp 3: Optimistic Locking - Kỹ thuật thay thế không dùng khóa CSDL

**Ý tưởng:** "Cứ cho rằng xung đột hiếm khi xảy ra", nên **không khóa**. Thay vào đó, thêm cột **`version`**. Khi ghi, kiểm tra xem version có còn như lúc đọc không. Nếu đã bị người khác sửa thì **ghi thất bại**, và ta thử lại.

```sql
ALTER TABLE products ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
```

**Cơ chế:**

| Bước | Khách A | Khách B |
|---|---|---|
| 1 | Đọc: quantity = 1, version = 5 | Đọc: quantity = 1, version = 5 |
| 2 | `UPDATE ... SET quantity = 0, version = 6 WHERE id = 5 AND version = 5` → **1 dòng, thành công** | |
| 3 | | `UPDATE ... WHERE id = 5 AND version = 5` → **0 dòng, thất bại** |

**Áp dụng trong Spring Boot:** Chỉ cần thêm `@Version`, Hibernate tự sinh câu `UPDATE ... WHERE version = ?` và tự tăng version.

```java
@Entity
public class Product {
    @Id
    private Long id;
    private int quantity;

    @Version
    private Long version;   // Hibernate quản lý tự động
    // getter, setter
}
```

Khi xung đột, Hibernate ném `ObjectOptimisticLockingFailureException`. Có thể thử lại tự động bằng Spring Retry:

```java
@Retryable(
    retryFor = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100))
@Transactional
public Order buy(Long userId, Long productId) {
    Product p = productRepository.findById(productId).orElseThrow();
    if (p.getQuantity() < 1) {
        throw new IllegalStateException("Hết hàng");
    }
    p.setQuantity(p.getQuantity() - 1);   // UPDATE kèm kiểm tra version
    return orderRepository.save(new Order(userId, productId));
}
```

> **Lưu ý:** Phải đặt `@Retryable` ở **bean ngoài** transaction để mỗi lần thử là một transaction mới. Thêm `@EnableRetry` và thư viện `spring-retry`.

**Ưu điểm:** Không giữ khóa nên **không chặn nhau, không deadlock**, rất hợp khi xung đột hiếm (sửa thông tin sản phẩm, sửa hồ sơ).
**Nhược điểm:** Khi xung đột nhiều (flash sale, trăm người mua 1 sản phẩm), phần lớn lần ghi thất bại và phải thử lại, gây lãng phí và tăng tải.

### So sánh nhanh 3 giải pháp

| Tiêu chí | Pessimistic | Atomic UPDATE | Optimistic |
|---|---|---|---|
| Cách làm | Khóa dòng khi đọc | Kiểm tra + trừ trong 1 lệnh | Kiểm tra `version` khi ghi |
| Có chặn nhau | Có | Rất ngắn | Không |
| Nguy cơ deadlock | Có | Thấp | Không |
| Khi xung đột nhiều | Ổn, xếp hàng chờ | **Tốt nhất** | Kém, phải retry nhiều |
| Khi xung đột ít | Hơi thừa | Tốt | **Tốt** |
| Độ phức tạp code | Trung bình | **Thấp** | Trung bình |
| Phù hợp | Logic phức tạp cần đọc rồi quyết định | Trừ tồn kho, cộng trừ số dư | Sửa dữ liệu ít tranh chấp |

## Deadlock - Rủi ro cần biết khi dùng locking

**Deadlock (khóa chết)** xảy ra khi 2 transaction **giữ khóa của nhau và cùng chờ nhau** mãi mãi. Ví dụ: A khóa sản phẩm 1 rồi chờ sản phẩm 2, trong khi B khóa sản phẩm 2 rồi chờ sản phẩm 1. InnoDB tự phát hiện và **hủy một transaction** (lỗi 1213, `Deadlock found`). Cách phòng tránh: **luôn khóa theo cùng một thứ tự** (sắp xếp theo id), giữ transaction ngắn, và **retry** khi gặp lỗi deadlock.

```java
@Transactional
public void buyMany(List<Long> productIds) {
    List<Long> sorted = productIds.stream().sorted().toList();   // luôn khóa theo thứ tự id tăng dần
    for (Long id : sorted) {
        productRepository.findByIdForUpdate(id);
        // ...
    }
}
```

## Tổng kết

- **Nguyên nhân gốc:** "đọc rồi mới ghi" ở hai bước tách rời.
- **Hướng giải quyết:** làm cho "kiểm tra + ghi" thành **một bước không thể chen ngang**, hoặc **phát hiện xung đột** khi ghi.

| Tình huống | Chọn |
|---|---|
| Trừ tồn kho, cộng trừ số dư | **Atomic UPDATE** (ưu tiên số 1) |
| Cần đọc, tính toán phức tạp rồi mới ghi, xung đột nhiều | **Pessimistic Locking** |
| Xung đột hiếm, muốn tránh khóa | **Optimistic Locking** |

**Cách nhớ nhanh:** Pessimistic là "khóa cửa trước khi vào", Atomic UPDATE là "kiểm tra và lấy hàng trong một động tác", Optimistic là "cứ làm, ai đến sau phát hiện bị lệch thì làm lại".
