# Phụ thuộc hàm

- **Định nghĩa:** Trong một bảng, thuộc tính Y **phụ thuộc hàm** vào X (ký hiệu **X → Y**) nếu với mỗi giá trị của X, luôn xác định được **duy nhất 1 giá trị** của Y.
- **Ví dụ trực quan:** Bảng `products`

| product_id | name | price |
|---|---|---|
| 1 | Áo thun | 150000 |
| 2 | Quần jean | 350000 |
| 3 | Mũ lưỡi trai | 150000 |

  - `product_id → name`: biết mã 1 thì chắc chắn ra "Áo thun", không thể ra tên khác. Tương tự `product_id → price`.
  - `price → product_id` **không đúng**: giá 150000 ứng với cả sản phẩm 1 và 3, nên biết giá thì không xác định được duy nhất sản phẩm.

> **Cách đọc:** "X xác định Y", hay "biết X thì suy ra được Y". X gọi là vế trái, Y là vế phải.

## Phụ thuộc hàm là nền tảng của mọi dạng chuẩn

Các dạng chuẩn (1NF, 2NF, 3NF, BCNF) thực chất đều là quy tắc về phụ thuộc hàm. 2NF cấm phụ thuộc bộ phận, 3NF cấm phụ thuộc bắc cầu, BCNF yêu cầu mọi phụ thuộc hàm đều xuất phát từ khóa. Hiểu phụ thuộc hàm, bạn sẽ biết bảng đang lỗi thiết kế ở đâu, vì sao dữ liệu bị lặp và cần tách bảng như thế nào.

## Các loại phụ thuộc hàm quan trọng

### a) Phụ thuộc hàm đầy đủ (Full Functional Dependency)

> Y phụ thuộc vào **toàn bộ** X, bỏ bớt bất kỳ cột nào của X thì không còn xác định được Y. Đây là loại phụ thuộc **tốt, mong muốn**.

- **Ví dụ:** Bảng `order_items(order_id, product_id, quantity)` có khóa chính (`order_id`, `product_id`).
  - `(order_id, product_id) → quantity` là phụ thuộc đầy đủ: chỉ biết đơn hàng thì chưa biết số lượng của sản phẩm nào, chỉ biết sản phẩm cũng không biết ở đơn nào. Phải có **cả cặp** mới xác định được.

### b) Phụ thuộc hàm bộ phận (Partial Functional Dependency) - vi phạm 2NF

> Y chỉ phụ thuộc vào **một phần** của khóa tổng hợp, chứ không cần cả khóa. Chỉ xảy ra khi khóa chính gồm từ 2 cột trở lên.

- **Ví dụ:** Bảng `order_items(order_id, product_id, product_name, quantity)`, khóa chính (`order_id`, `product_id`).
  - `product_id → product_name`: tên sản phẩm chỉ cần biết `product_id`, không liên quan đến `order_id`.
  - **Hậu quả:** Sản phẩm xuất hiện trong 100 đơn thì tên bị lặp 100 lần, sửa tên phải sửa 100 dòng.
  - **Cách sửa:** Tách `product_name` ra bảng `products`.

### c) Phụ thuộc hàm bắc cầu (Transitive Functional Dependency) - vi phạm 3NF

> X → Y và Y → Z (với Y **không phải khóa**) nên X → Z là suy ra **gián tiếp qua Y**.

- **Ví dụ:** Bảng `orders(order_id, customer_id, customer_name)`, khóa chính `order_id`.
  - `order_id → customer_id` và `customer_id → customer_name`, nên `order_id → customer_name` chỉ là hệ quả bắc cầu.
  - **Hậu quả:** Một khách có 50 đơn thì `customer_name` bị lặp 50 lần, đổi tên khách phải sửa nhiều dòng.
  - **Cách sửa:** Tách `customer_name` ra bảng `customers`, bảng `orders` chỉ giữ `customer_id`.

### d) Phụ thuộc hàm tầm thường (Trivial Functional Dependency)

> X → Y mà Y **nằm trong X** nên luôn đúng, không mang thông tin gì mới.

- **Ví dụ:**
  - `(order_id, product_id) → order_id` luôn đúng vì `order_id` đã nằm sẵn trong vế trái.
  - `product_id → product_id` cũng vậy.
  - Khi phân tích chuẩn hóa, ta **bỏ qua** loại này và chỉ quan tâm phụ thuộc hàm không tầm thường.

### Cách xác định phụ thuộc hàm trong thực tế - quy trình tư duy

1. **Liệt kê các cột** của bảng và xác định khóa chính.
2. **Với mỗi cột, tự hỏi:** "Biết giá trị của cột X thì có suy ra được duy nhất giá trị cột Y không?"
3. **Dựa vào quy tắc nghiệp vụ**, không dựa vào dữ liệu mẫu. Ví dụ dữ liệu hiện tại chưa có hai khách trùng tên, nhưng ngoài đời vẫn có, nên `customer_name → customer_id` là sai.
4. **Ghi ra dạng X → Y** cho từng cặp tìm được.
5. **Phân loại** từng phụ thuộc: cột không khóa phụ thuộc vào *cả khóa* (đạt), *một phần khóa* (vi phạm 2NF), hay *một cột không khóa khác* (vi phạm 3NF).
6. **Tách bảng** cho những phụ thuộc vi phạm, mỗi bảng chỉ nói về một đối tượng.

**Câu hỏi kiểm tra nhanh:** "Cột này mô tả cho cái gì: cho cả dòng, cho một phần khóa, hay cho một cột khác?"
