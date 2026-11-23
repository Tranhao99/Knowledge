# ON DELETE CASCADE / SET NULL / RESTRICT

## Vấn đề cần giải quyết

- Khi xóa hoặc sửa 1 dòng của **bảng cha** mà **bảng con** đang tham chiếu tới nó, MySQL cần biết xử lý các dòng con đó như thế nào?
- Ví dụ: xóa một người dùng khi người đó còn đơn hàng, các đơn hàng đó sẽ bị chặn lại, bị xóa theo, hay bị mất người mua?
- Cách xử lý được khai báo ngay khi tạo khóa ngoại: `ON DELETE ...` cho thao tác xóa, `ON UPDATE ...` cho thao tác sửa.

```sql
FOREIGN KEY (cột_con) REFERENCES bảng_cha(cột_cha)
    ON DELETE <hành vi>
    ON UPDATE <hành vi>
```

> Nếu không khai báo gì, MySQL (InnoDB) mặc định dùng `NO ACTION`, hoạt động giống `RESTRICT`.

## RESTRICT

- **Ví dụ:** Đơn hàng thuộc về người dùng. Không cho xóa người dùng khi họ còn đơn hàng.

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT
);
```

- **Hành vi của MySQL:**
  - **Chặn** thao tác xóa dòng cha nếu còn dòng con tham chiếu tới.
  - Báo lỗi `Cannot delete or update a parent row: a foreign key constraint fails`.
  - Muốn xóa được, phải xử lý (xóa hoặc chuyển) các dòng con trước.
- **Khi nào dùng RESTRICT:**
  - Dữ liệu quan trọng không được phép mất: người dùng, đơn hàng, hóa đơn, giao dịch.
  - Khi muốn bắt buộc lập trình viên xử lý dữ liệu con một cách có chủ đích.
  - Khi không chắc chọn gì, đây là lựa chọn **an toàn nhất**.

### SET NULL

- **Ví dụ:** Sản phẩm thuộc về một danh mục. Khi xóa danh mục, sản phẩm vẫn còn nhưng chưa có danh mục.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    category_id BIGINT NULL,
    CONSTRAINT fk_products_category
        FOREIGN KEY (category_id) REFERENCES categories(id)
        ON DELETE SET NULL
);
```

- **Hành vi của MySQL:**
  - Khi xóa dòng cha, MySQL tự động đặt cột khóa ngoại của các dòng con thành `NULL`.
  - Các dòng con **vẫn được giữ lại**.
  - Cột khóa ngoại **bắt buộc cho phép NULL**. Nếu khai báo `NOT NULL` thì không tạo được ràng buộc này.
- **Khi nào dùng SET NULL:**
  - Quan hệ **không bắt buộc**: sản phẩm có thể chưa thuộc danh mục nào.
  - Muốn giữ lại dữ liệu con dù cha đã bị xóa: bình luận của tài khoản đã xóa, đơn hàng của nhân viên đã nghỉ việc.

### CASCADE

- **Ví dụ:** Chi tiết đơn hàng chỉ có ý nghĩa khi có đơn hàng. Xóa đơn hàng thì xóa luôn các dòng chi tiết.

```sql
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    CONSTRAINT fk_items_order
        FOREIGN KEY (order_id) REFERENCES orders(id)
        ON DELETE CASCADE
);
```

- **Hành vi của MySQL:**
  - Khi xóa dòng cha, MySQL **tự động xóa tất cả dòng con** tham chiếu tới nó.
  - Việc xóa có thể **lan nhiều tầng** (xóa A kéo theo xóa B, rồi kéo theo xóa C).
  - Không cần viết thêm câu lệnh `DELETE` cho bảng con.
- **Khi nào dùng CASCADE:**
  - Dữ liệu con **phụ thuộc hoàn toàn** vào cha: chi tiết đơn hàng, ảnh của sản phẩm, mục trong giỏ hàng.
  - Dữ liệu con không có ý nghĩa nếu thiếu cha.
  - **Cẩn thận:** dùng sai có thể xóa mất lượng lớn dữ liệu chỉ bằng một lệnh. Không nên dùng cho dữ liệu quan trọng như đơn hàng của người dùng.

### ON UPDATE

- **Hoạt động tương tự như ON DELETE**, nhưng áp dụng khi **giá trị khóa chính của bảng cha bị sửa**. Các hành vi cũng gồm `RESTRICT`, `CASCADE`, `SET NULL`:
  - `CASCADE`: sửa khóa cha thì khóa ngoại ở các dòng con tự động sửa theo.
  - `RESTRICT`: không cho sửa khóa cha nếu còn dòng con tham chiếu.
  - `SET NULL`: sửa khóa cha thì khóa ngoại ở dòng con thành `NULL`.

```sql
CONSTRAINT fk_items_product
    FOREIGN KEY (product_sku) REFERENCES products(sku)
    ON UPDATE CASCADE
```

- **Một số trường hợp phổ biến:**
  1. **`ON UPDATE CASCADE`** khi khóa cha là **khóa tự nhiên** có thể bị đổi, như mã sản phẩm (`sku`), mã sách (`isbn`). Đổi mã ở bảng cha, các bảng con tự cập nhật theo, không bị lệch dữ liệu.
  2. **`ON UPDATE RESTRICT`** (mặc định) khi khóa cha là `id` **tự tăng**. Loại khóa này không nên và gần như không bao giờ bị sửa, nên chặn lại để tránh sửa nhầm.

## Tổng kết

| Hành vi | Khi xóa/sửa dòng cha | Dòng con | Ví dụ trong bán hàng |
|---|---|---|---|
| `RESTRICT` | Bị chặn, báo lỗi | Giữ nguyên | users - orders |
| `SET NULL` | Được phép | Khóa ngoại thành `NULL` | categories - products |
| `CASCADE` | Được phép | Bị xóa/sửa theo | orders - order_items |

**Liên hệ Spring Boot (JPA):** `ON DELETE CASCADE` là cơ chế của **database**, khác với `cascade = CascadeType.REMOVE` của JPA (do **ứng dụng** thực hiện). Nên chọn rõ một nơi để quản lý, tránh nhầm lẫn khi dữ liệu bị xóa ngoài ý muốn.

**Cách nhớ nhanh:** RESTRICT là "còn con thì không cho xóa cha", SET NULL là "xóa cha, con ở lại nhưng mất cha", CASCADE là "xóa cha thì xóa luôn con".

Khi bạn sẵn sàng, mình sẽ làm nội dung cuối của giai đoạn 2 là **Join**.
