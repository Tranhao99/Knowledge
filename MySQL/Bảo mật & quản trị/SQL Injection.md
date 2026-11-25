# Phòng chống SQL Injection

> **SQL Injection** là lỗ hổng xảy ra khi ứng dụng **ghép trực tiếp dữ liệu người dùng vào câu SQL**, khiến kẻ tấn công chèn mã SQL của chúng để đọc trộm, sửa hoặc xóa dữ liệu, hoặc vượt qua đăng nhập.

## 1. SQL Injection hoạt động như thế nào?

**Code có lỗ hổng** (ghép chuỗi):

```java
String sql = "SELECT * FROM users WHERE email = '" + email
           + "' AND password_hash = '" + password + "'";
```

Kẻ tấn công nhập vào ô email: `' OR '1'='1' -- `. Câu SQL thực tế thành:

```sql
SELECT * FROM users
WHERE email = '' OR '1'='1' -- ' AND password_hash = '...'
```

`'1'='1'` luôn đúng, còn `-- ` biến phần sau thành chú thích. Kết quả: **đăng nhập không cần mật khẩu**.

| Kiểu tấn công | Ví dụ đầu vào | Hậu quả |
|---|---|---|
| Vượt xác thực | `' OR '1'='1' -- ` | Đăng nhập không cần mật khẩu |
| Đọc trộm (UNION) | `' UNION SELECT email, password_hash FROM users -- ` | Lộ dữ liệu bảng khác |
| Blind injection | `' AND SLEEP(5) -- ` | Dò dữ liệu qua thời gian phản hồi |

## 2. Giải pháp cốt lõi: Prepared Statement (tham số hóa)

**Tách câu SQL và dữ liệu.** Câu SQL gửi trước với chỗ trống `?`, dữ liệu gửi sau như **giá trị thuần túy**. Dù chứa `'` hay `OR`, nó cũng chỉ là chuỗi ký tự, không bao giờ trở thành lệnh.

```java
String sql = "SELECT * FROM users WHERE email = ?";

try (PreparedStatement ps = connection.prepareStatement(sql)) {
    ps.setString(1, email);
    ResultSet rs = ps.executeQuery();
}
```

Với đầu vào `' OR '1'='1' -- `, database chỉ tìm user có email **đúng bằng chuỗi đó** và không thấy gì.

## 3. Áp dụng trong Spring Boot

**a) Spring Data JPA: method name** (an toàn mặc định)

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<Product> findByNameContainingIgnoreCase(String keyword);
}
```

**b) `@Query` với tham số có tên**

```java
@Query("SELECT p FROM Product p WHERE p.name LIKE %:keyword% AND p.price <= :maxPrice")
List<Product> search(@Param("keyword") String keyword,
                     @Param("maxPrice") BigDecimal maxPrice);

@Query(value = "SELECT * FROM products WHERE category_id = :categoryId", nativeQuery = true)
List<Product> findByCategory(@Param("categoryId") Long categoryId);
```

**c) `NamedParameterJdbcTemplate`**

```java
String sql = "SELECT * FROM orders WHERE user_id = :userId AND status = :status";
List<Order> orders = namedJdbcTemplate.query(
        sql, Map.of("userId", userId, "status", status), orderRowMapper);
```

## 4. Những cách viết vẫn bị SQL Injection dù dùng Spring Boot

**Sai lầm 1: Ghép chuỗi trong `JdbcTemplate` hoặc `EntityManager`**

```java
// SAI
jdbcTemplate.query("SELECT * FROM products WHERE name = '" + name + "'", rowMapper);
em.createNativeQuery("SELECT * FROM users WHERE email = '" + email + "'");

// ĐÚNG
jdbcTemplate.query("SELECT * FROM products WHERE name = ?", rowMapper, name);
```

**Sai lầm 2: Cho người dùng truyền thẳng tên cột hoặc hướng sắp xếp vào `ORDER BY`**

Tham số `?` **chỉ dùng cho giá trị**, không dùng cho tên cột, tên bảng, `ASC/DESC`. Với những phần này phải dùng **whitelist (danh sách cho phép)**:

```java
private static final Set<String> ALLOWED_SORT = Set.of("name", "price", "createdAt");

public List<Product> list(String sortBy, String direction) {
    String column = ALLOWED_SORT.contains(sortBy) ? sortBy : "createdAt";
    Sort.Direction dir = "desc".equalsIgnoreCase(direction)
            ? Sort.Direction.DESC : Sort.Direction.ASC;
    return productRepository.findAll(Sort.by(dir, column));
}
```

**Sai lầm 3: Ghép wildcard `LIKE` vào câu SQL.** Hãy thêm `%` vào **giá trị**:

```java
ps.setString(1, "%" + keyword + "%");
```

**Sai lầm 4: Dùng `${}` thay vì `#{}` trong MyBatis**

```xml
<!-- SAI: ghép chuỗi trực tiếp -->
<select id="find">SELECT * FROM users WHERE email = '${email}'</select>
<!-- ĐÚNG: tham số hóa -->
<select id="find">SELECT * FROM users WHERE email = #{email}</select>
```

## 5. Các lớp phòng thủ bổ sung

Prepared statement là lớp quan trọng nhất, nên kết hợp thêm:

**a) Validation đầu vào** (giảm bề mặt tấn công, nhưng **không thay thế** tham số hóa)

```java
public record RegisterRequest(
    @NotBlank @Email @Size(max = 255) String email,
    @NotBlank @Size(min = 8, max = 64) String password
) {}
```

**b) User database quyền tối thiểu.** Nếu bị tấn công, thiệt hại cũng bị giới hạn.

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON shop_db.* TO 'shop_app'@'10.0.0.%';
```

**c) Không để lộ lỗi SQL ra ngoài.** Tên bảng, tên cột trong thông báo lỗi giúp kẻ tấn công dò cấu trúc.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<Map<String, String>> handleDb(DataAccessException ex) {
        log.error("Database error", ex);
        return ResponseEntity.status(500).body(Map.of("message", "Đã xảy ra lỗi hệ thống"));
    }
}
```

**d) Các biện pháp khác:**
- Ưu tiên **JPA**, hạn chế viết SQL thủ công.
- **Hash mật khẩu** bằng Bcrypt để lộ dữ liệu cũng không dùng được ngay.
- **Cập nhật** Spring, Hibernate, MySQL Connector thường xuyên.
- **Quét bảo mật** định kỳ (OWASP ZAP, SonarQube).

## 6. Checklist rà soát code

- [ ] Không ghép chuỗi (`+`, `String.format`) vào câu SQL.
- [ ] Mọi giá trị từ người dùng đều đi qua tham số (`?`, `:name`, `@Param`).
- [ ] Tên cột, tên bảng, hướng sắp xếp do người dùng chọn đều qua **whitelist**.
- [ ] Không dùng `${}` trong MyBatis cho dữ liệu người dùng.
- [ ] User database của ứng dụng không có quyền DDL.
- [ ] Lỗi SQL không bị trả nguyên văn về client.

## Tổng kết

| Nguyên tắc | Cách làm |
|---|---|
| Tách lệnh và dữ liệu | Prepared Statement, `@Param`, `NamedParameterJdbcTemplate` |
| Tên cột, tên bảng, `ORDER BY` | Không dùng `?`, phải dùng whitelist |
| Giới hạn thiệt hại | User database quyền tối thiểu |
| Không tiết lộ thông tin | Ẩn lỗi SQL chi tiết |
| Nhiều lớp bảo vệ | Validation + phân quyền + hash mật khẩu + cập nhật thư viện |

**Cách nhớ nhanh:** Đừng **tin dữ liệu người dùng** và đừng **ghép chuỗi vào SQL**. Dữ liệu luôn đi qua tham số, còn thứ gì không đi qua tham số được (tên cột, tên bảng) thì kiểm tra bằng danh sách cho phép.
