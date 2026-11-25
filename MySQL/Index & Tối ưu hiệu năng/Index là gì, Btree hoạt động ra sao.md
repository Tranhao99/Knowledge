# Index và B-Tree

## Tại sao cần index?

Khi không có index, để tìm một dòng MySQL phải **đọc lần lượt toàn bộ bảng** (full table scan). Bảng càng lớn thì càng chậm: 10 triệu dòng nghĩa là kiểm tra 10 triệu lần. Index giống **mục lục của cuốn sách**: thay vì lật từng trang, bạn tra mục lục để nhảy thẳng đến trang cần tìm. Index giúp `WHERE`, `JOIN`, `ORDER BY` nhanh hơn rất nhiều, nhưng đổi lại tốn thêm dung lượng và làm thao tác ghi chậm hơn.

## Minh họa trực quan

Bảng `users` có 1.000.000 dòng, cần tìm theo email:

```sql
SELECT * FROM users WHERE email = 'an@gmail.com';
```

- **Không có index:** MySQL quét cả 1.000.000 dòng. `EXPLAIN` hiển thị `type = ALL`.
- **Có index:** MySQL chỉ cần đọc khoảng 3-4 vị trí để tìm ra dòng. `EXPLAIN` hiển thị `type = ref`.

```sql
CREATE INDEX idx_users_email ON users(email);

EXPLAIN SELECT * FROM users WHERE email = 'an@gmail.com';
```

Từ hàng triệu lần so sánh xuống chỉ còn vài lần đọc.

## Cấu trúc B-Tree - nền tảng của hầu hết index trong MySQL

**B-Tree (cây cân bằng)** gồm 3 loại nút: **root** (gốc), **branch** (nhánh trung gian) và **leaf** (lá). Các khóa luôn được **sắp xếp sẵn** theo thứ tự. Khi tìm kiếm, MySQL đi từ gốc xuống, mỗi tầng so sánh để chọn đúng nhánh và loại bỏ phần lớn dữ liệu, cho đến khi tới lá.

**Các tính chất quan trọng:**
- **Cân bằng:** mọi lá nằm cùng độ sâu nên thời gian tìm kiếm ổn định.
- **Rất thấp:** mỗi nút là một page 16KB chứa hàng trăm khóa, nên vài triệu dòng chỉ cao khoảng 3-4 tầng, tức 3-4 lần đọc.
- **Lá nối với nhau theo thứ tự:** nên truy vấn khoảng (`BETWEEN`, `>`, `<`) và `ORDER BY` rất nhanh.

**Hệ quả khi viết truy vấn:**
- Hiệu quả với: `=`, khoảng giá trị, `ORDER BY`, `LIKE 'abc%'`.
- Không hiệu quả với: `LIKE '%abc'` hoặc bọc cột trong hàm như `WHERE YEAR(order_date) = 2026`.

**Chi phí:** `INSERT`, `UPDATE`, `DELETE` phải cập nhật cả cây, đôi khi làm tách page, nên ghi sẽ chậm hơn.

```
                 [ 50 | 100 ]                 ← root
              /       |        \
        [10|30]    [60|80]    [120|150]       ← branch
        /  |  \    /  |  \     /   |   \
      [..]→[..]→[..]→[..]→[..]→[..]→[..]      ← leaf, nối nhau theo thứ tự
```

Tìm giá trị 80: gốc (50 < 80 < 100) → nhánh giữa (60|80) → lá chứa 80.

## Cấu trúc thực tế trong InnoDB: Clustered Index và Secondary Index

**Clustered Index (chỉ mục phân cụm):**
- Chính là **bảng dữ liệu**, được tổ chức thành B-Tree theo **khóa chính**. Nút lá chứa **toàn bộ dòng dữ liệu**.
- Mỗi bảng chỉ có **1** clustered index. Nếu không khai báo khóa chính, InnoDB dùng cột `UNIQUE NOT NULL` đầu tiên, nếu không có nữa thì tự tạo cột ẩn.

**Secondary Index (chỉ mục phụ):**
- Là mọi index khác do bạn tạo. Nút lá chỉ chứa **giá trị cột được đánh index + giá trị khóa chính**, không chứa cả dòng.
- Tìm bằng secondary index cần **2 bước**: tìm trong index phụ để lấy khóa chính, rồi tìm tiếp trong clustered index để lấy dòng đầy đủ.

**Covering index:** Nếu câu truy vấn chỉ cần các cột nằm trong index (kể cả khóa chính) thì **bỏ được bước 2**, nhanh hơn. `EXPLAIN` hiển thị `Using index`.

**Hệ quả thiết kế:**
- Khóa chính nên **ngắn và tăng dần** (`BIGINT AUTO_INCREMENT`), vì mọi secondary index đều chứa bản sao khóa chính.
- `UUID` ngẫu nhiên làm chèn rải rác khắp cây, gây tách page, chậm và tốn dung lượng.

```
Clustered (PK id):        [id=1 | toàn bộ dòng] → [id=2 | toàn bộ dòng] → ...
Secondary idx_email:      [email='an@..' | id=1] → [email='binh@..' | id=2] → ...
```

```sql
-- 2 bước: idx_users_email lấy id, rồi quay lại clustered lấy cả dòng
SELECT * FROM users WHERE email = 'an@gmail.com';

-- Covering index: chỉ cần id và email, không cần quay lại bảng
SELECT id, email FROM users WHERE email = 'an@gmail.com';
```

## Tổng kết

| Khái niệm | Ý chính |
|---|---|
| Index | Mục lục giúp tránh quét toàn bảng |
| B-Tree | Cây cân bằng, sắp xếp sẵn, chỉ 3-4 tầng cho hàng triệu dòng |
| Clustered index | Chính là bảng, sắp theo khóa chính, mỗi bảng 1 cái |
| Secondary index | Lá chứa khóa chính, tìm xong phải quay lại clustered |
| Covering index | Index chứa đủ cột cần lấy, không cần quay lại bảng |

**Cách nhớ nhanh:** Clustered index là "cuốn sách", secondary index là "mục lục tra từ khóa", và mục lục chỉ cho bạn **số trang** (khóa chính) để lật đến nội dung.

Khi bạn sẵn sàng, mình có thể làm tiếp **cách tạo và chọn index** (single, composite, leftmost prefix) hoặc **đọc EXPLAIN**.
