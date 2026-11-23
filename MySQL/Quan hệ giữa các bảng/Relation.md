# Quan hệ giữa các bảng

> **Quan hệ giữa các bảng** cho biết dữ liệu ở bảng này liên kết với dữ liệu ở bảng kia như thế nào. Quan hệ được tạo bằng **khóa ngoại (foreign key)** và chia thành 3 loại: 1-1, 1-N, N-N. Nhờ đó dữ liệu không bị lặp lại và dễ quản lý hơn.

## 1-1

- **Định nghĩa:** Một dòng ở bảng A liên kết với **đúng 1 dòng** ở bảng B và ngược lại.
- **Ví dụ:** Mỗi người dùng (`users`) có đúng một hồ sơ cá nhân (`user_profiles`), và mỗi hồ sơ chỉ thuộc về một người dùng.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    full_name VARCHAR(100),
    address VARCHAR(255),
    avatar_url VARCHAR(255),
    CONSTRAINT fk_profile_user
        FOREIGN KEY (user_id) REFERENCES users(id)
);
```

Cách làm: cột `user_id` vừa là **khóa chính** vừa là **khóa ngoại**, nên mỗi người dùng chỉ có tối đa 1 hồ sơ. Cách khác là để khóa ngoại là cột riêng và thêm `UNIQUE`.

- **Khi nào nên tách bảng 1-1 thay vì gộp chung?**
  - Dữ liệu **ít khi dùng hoặc rất nặng** (ảnh, mô tả dài), tách ra để bảng chính nhẹ và truy vấn nhanh hơn.
  - Dữ liệu **nhạy cảm** cần phân quyền riêng (thông tin thanh toán, CCCD).
  - Dữ liệu **tùy chọn**, nhiều người để trống, gộp chung sẽ có nhiều cột NULL.
  - Ngược lại, nếu hai phần luôn dùng cùng nhau và ít cột thì **gộp chung** cho đơn giản.

## 1-N

- **Định nghĩa:** Một dòng ở bảng A liên kết với **nhiều dòng** ở bảng B, nhưng mỗi dòng ở bảng B chỉ thuộc về **1 dòng** ở bảng A. Đây là loại quan hệ phổ biến nhất.
- **Ví dụ:** Một người dùng có nhiều đơn hàng, nhưng mỗi đơn hàng chỉ thuộc về một người dùng.

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id) REFERENCES users(id)
);
```

Dữ liệu mẫu:

| orders.id | orders.user_id |
|---|---|
| 1 | 1 |
| 2 | 1 |
| 3 | 2 |

Người dùng 1 có 2 đơn (id 1 và 2), người dùng 2 có 1 đơn (id 3).

- **Cách thiết kế:**
  - Đặt khóa ngoại ở bảng phía **N** (bảng `orders`), trỏ về khóa chính của bảng phía **1** (bảng `users`).
  - Thêm `NOT NULL` nếu bản ghi bắt buộc phải có "cha" (đơn hàng bắt buộc có người mua).
  - Các ví dụ khác trong bán hàng: 1 danh mục có nhiều sản phẩm, 1 đơn hàng có nhiều dòng chi tiết.

## N-N

- **Định nghĩa:** Một dòng ở bảng A liên kết với **nhiều dòng** ở bảng B và ngược lại. MySQL không tạo trực tiếp quan hệ này được, nên phải dùng thêm một **bảng trung gian** để tách thành 2 quan hệ 1-N.
- **Ví dụ:** Một đơn hàng có nhiều sản phẩm, và một sản phẩm có thể nằm trong nhiều đơn hàng. Bảng trung gian là `order_items`.

```sql
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_item_order
        FOREIGN KEY (order_id) REFERENCES orders(id),
    CONSTRAINT fk_item_product
        FOREIGN KEY (product_id) REFERENCES products(id)
);
```

Dữ liệu mẫu:

| order_id | product_id | quantity | unit_price |
|---|---|---|---|
| 1 | 1 | 2 | 150000 |
| 1 | 3 | 1 | 800000 |
| 2 | 1 | 1 | 150000 |

Đơn 1 có 2 sản phẩm (1 và 3), sản phẩm 1 xuất hiện trong 2 đơn (1 và 2).

- **Cách thiết kế:**
  - Bảng trung gian chứa **2 khóa ngoại**, thường dùng cặp đó làm **khóa chính tổng hợp** để không bị trùng.
  - Bảng trung gian có thể chứa **thêm cột riêng** của mối quan hệ, như `quantity`, `unit_price`.
  - Lưu `unit_price` tại thời điểm mua để sau này đổi giá sản phẩm không làm sai đơn hàng cũ.

## Tổng kết

| Quan hệ | Cách tạo | Ví dụ trong bán hàng |
|---|---|---|
| 1-1 | Khóa ngoại + `UNIQUE` (hoặc dùng luôn làm khóa chính) | users - user_profiles |
| 1-N | Khóa ngoại đặt ở bảng phía N | users - orders |
| N-N | Bảng trung gian chứa 2 khóa ngoại | orders - products (qua order_items) |

**Liên hệ Spring Boot (JPA):** 1-1 ứng với `@OneToOne`, 1-N ứng với `@OneToMany` và `@ManyToOne`, N-N ứng với `@ManyToMany` (hoặc tách bảng trung gian thành entity riêng khi có thêm cột như `quantity`).

**Cách nhớ nhanh:** Muốn biết là quan hệ gì, hãy hỏi 2 chiều: "Một A có nhiều B không?" và "Một B có nhiều A không?". Cả hai cùng "không" là 1-1, một "có" một "không" là 1-N, cả hai cùng "có" là N-N.

Khi bạn sẵn sàng, mình sẽ làm tiếp nội dung **Delete-update mode** (RESTRICT, CASCADE, SET NULL...) và **Join**.
