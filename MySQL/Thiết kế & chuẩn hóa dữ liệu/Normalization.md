# Normalization

> **Normalization (chuẩn hóa dữ liệu)** là quá trình tổ chức lại các bảng để **loại bỏ dữ liệu lặp** và **tránh lỗi khi thêm, sửa, xóa**. Cách làm: tách bảng "ôm đồm" thành nhiều bảng nhỏ, mỗi bảng chỉ nói về một đối tượng, rồi nối lại bằng khóa ngoại.

## Vấn đề gốc: tại sao không nhét tất cả mọi thứ vào một bảng?

**Ví dụ:** Bảng `orders_all` chứa mọi thứ, khóa chính là (`order_id`, `product_id`).

| order_id | customer_id | customer_name | product_id | product_name | price | quantity |
|---|---|---|---|---|---|---|
| 1 | 10 | Nguyễn An | 1 | Áo thun | 150000 | 2 |
| 1 | 10 | Nguyễn An | 3 | Mũ lưỡi trai | 90000 | 1 |
| 2 | 10 | Nguyễn An | 1 | Áo thun | 150000 | 1 |
| 3 | 11 | Trần Bình | 2 | Quần jean | 350000 | 1 |

Nhìn qua thì tiện, nhưng "Nguyễn An" và "Áo thun, 150000" bị lặp lại nhiều lần. Từ đó sinh ra 3 lỗi kinh điển:

- **Insertion Anomaly (lỗi khi thêm):** Muốn thêm sản phẩm mới chưa ai mua hoặc khách mới chưa đặt hàng thì không được, vì khóa chính cần `order_id`. Phải nhét dữ liệu giả hoặc để trống.
- **Update Anomaly (lỗi khi sửa):** Đổi giá "Áo thun" thì phải sửa **mọi dòng** có áo thun. Sửa sót một dòng thì cùng một sản phẩm có hai giá khác nhau, dữ liệu mâu thuẫn.
- **Deletion Anomaly (lỗi khi xóa):** Xóa đơn hàng số 3 (đơn duy nhất của Trần Bình, đơn duy nhất có Quần jean) thì **mất luôn thông tin** khách Trần Bình và sản phẩm Quần jean.

## Các dạng chuẩn hóa

Mỗi dạng chuẩn là một mức "sạch" của thiết kế. Phải đạt dạng chuẩn trước rồi mới xét dạng chuẩn sau: 1NF → 2NF → 3NF. Ở thực tế, bán hàng thường dừng ở **3NF** là đủ tốt.

### Dạng chuẩn 1NF

- **Đặc điểm:**
  - Mỗi ô chỉ chứa **một giá trị duy nhất** (nguyên tử), không có danh sách trong một ô.
  - Không có **nhóm cột lặp** (như `product1`, `product2`, `product3`).
  - Mỗi dòng phải **phân biệt được** với dòng khác (có khóa chính).
- **Ví dụ vi phạm:** Một ô chứa nhiều sản phẩm.

| order_id | customer_name | products |
|---|---|---|
| 1 | Nguyễn An | Áo thun, Mũ lưỡi trai |

  Hậu quả: muốn tìm đơn có "Áo thun" hay đếm số sản phẩm của đơn đều rất khó.
- **Cách sửa:** Mỗi sản phẩm một dòng.

| order_id | customer_name | product_name |
|---|---|---|
| 1 | Nguyễn An | Áo thun |
| 1 | Nguyễn An | Mũ lưỡi trai |

### Dạng chuẩn 2NF

- **Đặc điểm:**
  - Đã đạt 1NF.
  - **Không có phụ thuộc hàm bộ phận**: mọi cột không phải khóa phải phụ thuộc vào **toàn bộ** khóa chính.
  - Chỉ cần xét khi khóa chính gồm **2 cột trở lên**. Bảng có khóa chính 1 cột và đã đạt 1NF thì tự động đạt 2NF.
- **Ví dụ vi phạm:** Bảng `orders_all` ở trên, khóa chính (`order_id`, `product_id`).
  - `product_id → product_name, price`: chỉ phụ thuộc **một phần** khóa.
  - `order_id → customer_id, customer_name`: cũng chỉ phụ thuộc **một phần** khóa.
  - Chỉ `quantity` là phụ thuộc đầy đủ vào cả cặp.
- **Cách sửa:** Tách theo từng phần khóa.

```sql
-- Thông tin sản phẩm chỉ phụ thuộc product_id
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

-- Thông tin đơn hàng chỉ phụ thuộc order_id
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    customer_name VARCHAR(100) NOT NULL
);

-- Chỉ giữ cột phụ thuộc cả cặp khóa
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

### Dạng chuẩn 3NF

- **Đặc điểm:**
  - Đã đạt 2NF.
  - **Không có phụ thuộc hàm bắc cầu**: cột không phải khóa **không được phụ thuộc vào một cột không phải khóa khác**.
  - Nói gọn: mọi cột không phải khóa chỉ được phụ thuộc vào **khóa, toàn bộ khóa và không gì khác ngoài khóa**.
- **Ví dụ vi phạm:** Bảng `orders` vừa tách ở 2NF.

| order_id | customer_id | customer_name |
|---|---|---|
| 1 | 10 | Nguyễn An |
| 2 | 10 | Nguyễn An |

  - `order_id → customer_id` và `customer_id → customer_name`, nên `customer_name` phụ thuộc **bắc cầu** vào `order_id`.
  - Hậu quả: khách có 50 đơn thì tên bị lặp 50 lần, đổi tên phải sửa 50 dòng.
- **Cách sửa:** Tách thông tin khách hàng ra bảng riêng.

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

## Kết quả sau khi chuẩn hóa

Bảng `orders_all` ban đầu đã được tách thành 4 bảng, mỗi bảng nói về một đối tượng:

| Bảng | Nội dung | Quan hệ |
|---|---|---|
| `customers` | Thông tin khách hàng | 1 khách - N đơn hàng |
| `orders` | Thông tin đơn hàng | 1 đơn - N dòng chi tiết |
| `products` | Thông tin sản phẩm | 1 sản phẩm - N dòng chi tiết |
| `order_items` | Sản phẩm nào, số lượng bao nhiêu trong đơn nào | Bảng trung gian N-N |

3 lỗi ban đầu đã được giải quyết:
- **Thêm:** thêm sản phẩm hay khách mới thoải mái, không cần có đơn hàng.
- **Sửa:** đổi giá hay tên ở **một nơi duy nhất**.
- **Xóa:** xóa đơn hàng không làm mất thông tin khách và sản phẩm.

## Tổng kết

| Dạng chuẩn | Yêu cầu | Cách nhận biết vi phạm |
|---|---|---|
| 1NF | Mỗi ô một giá trị, không nhóm cột lặp | Có danh sách trong một ô |
| 2NF | Không phụ thuộc bộ phận | Cột phụ thuộc một phần khóa tổng hợp |
| 3NF | Không phụ thuộc bắc cầu | Cột phụ thuộc vào cột không phải khóa |

**Lưu ý thực tế:** Chuẩn hóa càng cao thì càng nhiều bảng, truy vấn phải `JOIN` nhiều hơn. Với dữ liệu cần đọc cực nhanh (báo cáo, thống kê) đôi khi người ta cố ý **phi chuẩn hóa** (denormalization) một phần, nhưng chỉ nên làm khi đã hiểu rõ lý do.

**Cách nhớ nhanh:** 1NF là "mỗi ô một giá trị", 2NF là "phụ thuộc vào toàn bộ khóa", 3NF là "phụ thuộc vào khóa và không gì khác ngoài khóa".
