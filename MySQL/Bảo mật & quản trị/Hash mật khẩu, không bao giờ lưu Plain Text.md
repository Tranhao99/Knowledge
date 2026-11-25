# Hash mật khẩu, không bao giờ lưu Plain Text

## Tại sao tuyệt đối không lưu mật khẩu dưới dạng Plain Text?

- **Lộ database là lộ tất cả:** chỉ cần một lỗ hổng SQL Injection, một bản backup bị rò rỉ hoặc một nhân viên xấu là toàn bộ mật khẩu bị đọc được ngay.
- **Hậu quả lan sang dịch vụ khác:** nhiều người dùng chung một mật khẩu cho email, ngân hàng, mạng xã hội. Lộ ở hệ thống của bạn là nguy cơ cho cả các tài khoản đó.
- **Vi phạm pháp lý và mất uy tín:** nhiều quy định về bảo vệ dữ liệu coi đây là lỗi nghiêm trọng, và ngay cả quản trị viên cũng không có lý do chính đáng để xem mật khẩu của người dùng.

```sql
-- Sai hoàn toàn: ai mở bảng cũng đọc được mật khẩu
INSERT INTO users (email, password) VALUES ('an@gmail.com', '123456');
```

## Hashing là gì? Khác biệt cốt lõi với Encryption

**Hashing (băm)** biến một chuỗi bất kỳ thành một chuỗi cố định gọi là hash, theo **một chiều**: từ mật khẩu ra hash thì dễ, nhưng từ hash không thể tính ngược lại ra mật khẩu. Cùng đầu vào luôn cho cùng đầu ra. Khi đăng nhập, ta băm mật khẩu người dùng nhập vào rồi **so sánh hash**, không bao giờ cần biết mật khẩu gốc.

| Tiêu chí | Hashing (băm) | Encryption (mã hóa) |
|---|---|---|
| Chiều | **Một chiều**, không giải ngược được | **Hai chiều**, giải mã được bằng khóa |
| Có cần khóa | Không | Có (khóa bí mật) |
| Mục đích | Kiểm tra dữ liệu có khớp hay không | Giữ bí mật rồi đọc lại khi cần |
| Dùng cho | **Mật khẩu** | Số thẻ, tin nhắn, dữ liệu cần đọc lại |
| Nếu lộ khóa | Không có khóa để lộ | Giải mã được toàn bộ dữ liệu |

> **Kết luận:** Mật khẩu không cần được đọc lại, chỉ cần kiểm tra khớp hay không, nên dùng **hashing**, không dùng encryption.

## Vì sao không dùng hàm băm đơn giản như MD5, SHA-1, SHA-256?

Các hàm này được thiết kế để **chạy rất nhanh**, và đó chính là điểm yếu. Một card đồ họa có thể thử hàng tỷ mật khẩu mỗi giây, nên mật khẩu yếu bị dò ra trong vài phút. Chúng cũng **không có salt**: hai người cùng mật khẩu sẽ có cùng hash, và kẻ tấn công dùng **rainbow table** (bảng hash tính sẵn) để tra ngược gần như tức thì. Riêng MD5 và SHA-1 còn bị phá về mặt mật mã. Mật khẩu cần thuật toán **cố tình chậm** và **có salt**.

## Bcrypt - Thuật toán được thiết kế riêng cho mật khẩu

### a) Salt tự động - giải quyết vấn đề Rainbow Table

**Salt** là một chuỗi ngẫu nhiên được thêm vào mật khẩu **trước khi băm**. Bcrypt tự sinh salt mới cho mỗi lần băm và lưu luôn trong chuỗi kết quả, nên bạn không cần quản lý riêng.

- Hai người cùng mật khẩu `123456` sẽ có **hai hash hoàn toàn khác nhau**.
- Rainbow table vô dụng vì kẻ tấn công phải tính lại bảng cho **từng salt**.

```java
encoder.encode("123456");   // $2a$10$N9qo8uLOickgx2ZMRZoMye...
encoder.encode("123456");   // $2a$10$7EqJtq98hPqEX7fNZaFWoO... (khác hẳn)
```

### b) Cost Factor - cố tình làm chậm để tránh brute-force

**Cost factor** (còn gọi là strength hay work factor) quy định số vòng lặp của thuật toán là **2 mũ cost**. Mỗi lần tăng cost thêm 1 thì thời gian băm **gấp đôi**.

| Cost | Số vòng | Thời gian băm (ước chừng) |
|---|---|---|
| 10 (mặc định) | 1.024 | khoảng 50-100 ms |
| 12 | 4.096 | khoảng 250-400 ms |
| 14 | 16.384 | khoảng 1 giây trở lên |

Với người dùng thật, chờ vài chục mili giây khi đăng nhập là không đáng kể. Nhưng với kẻ thử hàng tỷ mật khẩu, mỗi lần thử đều chậm nên **brute-force trở nên không khả thi**. Khi phần cứng mạnh lên, ta chỉ cần tăng cost. Nên chọn mức khiến mỗi lần băm mất khoảng **100-500 ms** trên server của bạn.

### c) Cấu trúc chuỗi hash Bcrypt

Chuỗi kết quả luôn dài **60 ký tự**, gồm 4 phần:

```
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
└┬┘└┬┘└──────────┬──────────┘└──────────────┬────────────────┘
 │  │             │                          │
 │  │             │                          └ Hash (31 ký tự)
 │  │             └ Salt (22 ký tự)
 │  └ Cost factor (10)
 └ Phiên bản thuật toán (2a)
```

Vì salt và cost nằm sẵn trong chuỗi, khi kiểm tra mật khẩu thư viện tự đọc ra rồi băm lại cho khớp. Cột lưu mật khẩu chỉ cần `VARCHAR(60)`, nhưng nên để `VARCHAR(255)` để sau này đổi sang thuật toán khác vẫn đủ chỗ.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL
);
```

> **Giới hạn:** Bcrypt chỉ xử lý **tối đa 72 byte** đầu của mật khẩu, phần dư bị bỏ qua. Nên giới hạn độ dài mật khẩu khi đăng ký (ví dụ 8 đến 64 ký tự).

## Quy trình đăng ký và đăng nhập - nơi Bcrypt luôn hoạt động (luôn ở tầng ứng dụng, không phải SQL)

Việc băm và kiểm tra **luôn làm ở tầng ứng dụng** (Spring Boot), rồi mới lưu hash xuống database. Database chỉ thấy một chuỗi 60 ký tự, **không bao giờ nhận mật khẩu gốc**.

```
Đăng ký:   mật khẩu gốc → [Spring: encode] → hash → INSERT vào DB
Đăng nhập: mật khẩu nhập → [Spring: matches(nhập, hash lấy từ DB)] → đúng / sai
```

**Cấu hình bean:**

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);   // cost = 12
    }
}
```

**Đăng ký: băm trước khi lưu**

```java
@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public void register(String email, String rawPassword) {
        if (userRepository.existsByEmail(email)) {
            throw new IllegalArgumentException("Email đã được sử dụng");
        }

        User user = new User();
        user.setEmail(email);
        user.setPasswordHash(passwordEncoder.encode(rawPassword));   // băm tại đây
        userRepository.save(user);
    }
}
```

**Đăng nhập: so sánh bằng `matches`**

```java
public User login(String email, String rawPassword) {
    User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new BadCredentialsException("Email hoặc mật khẩu không đúng"));

    if (!passwordEncoder.matches(rawPassword, user.getPasswordHash())) {
        throw new BadCredentialsException("Email hoặc mật khẩu không đúng");
    }
    return user;
}
```

**Những điều cần nhớ:**
- **Không bao giờ** tự so sánh `encode(rawPassword).equals(hash)`. Mỗi lần `encode` ra salt khác nhau nên luôn sai. Phải dùng `matches`.
- **Không ghi log mật khẩu gốc**, kể cả log lỗi hoặc log request.
- Không bao giờ trả trường `passwordHash` ra API: dùng DTO riêng hoặc `@JsonIgnore`.
- Nếu dùng Spring Security đầy đủ, chỉ cần khai báo bean `PasswordEncoder` và `UserDetailsService`, framework tự gọi `matches` khi xác thực.

## Lưu ý bảo mật quan trọng: Thông báo lỗi mơ hồ khi đăng nhập sai

Luôn trả về **cùng một thông báo chung** dù sai email hay sai mật khẩu, ví dụ: "Email hoặc mật khẩu không đúng".

| Cách làm | Hậu quả |
|---|---|
| "Email không tồn tại" / "Sai mật khẩu" | Kẻ tấn công **dò được email nào có tài khoản** (user enumeration), rồi chỉ cần tập trung đoán mật khẩu |
| "Email hoặc mật khẩu không đúng" | Không tiết lộ gì thêm |

**Các biện pháp đi kèm:**
- **Timing attack:** nếu email không tồn tại mà trả lời nhanh hơn rõ rệt (vì bỏ qua bước băm), kẻ tấn công vẫn đo thời gian để đoán. Cách giảm thiểu là vẫn chạy một lần `matches` giả khi không tìm thấy user. Spring Security đã xử lý sẵn điều này.
- **Giới hạn số lần thử** (rate limiting) và **khóa tạm** tài khoản sau nhiều lần sai.
- Cùng nguyên tắc cho chức năng **quên mật khẩu**: luôn báo "Nếu email tồn tại, chúng tôi đã gửi hướng dẫn".

## So sánh Bcrypt và các lựa chọn hiện đại khác

| Tiêu chí | Bcrypt | PBKDF2 | Scrypt | Argon2id |
|---|---|---|---|---|
| Năm ra đời | 1999 | 2000 | 2009 | 2015 (thắng cuộc thi PHC) |
| Chống brute-force | Tốt (chậm theo CPU) | Khá (chỉ chậm theo CPU) | Rất tốt (tốn cả RAM) | **Rất tốt nhất** (tốn RAM và CPU, tùy chỉnh) |
| Chống tấn công bằng GPU/ASIC | Khá | Yếu hơn | Tốt | **Tốt nhất** |
| Tham số điều chỉnh | Cost | Số vòng lặp | CPU, RAM, song song | Thời gian, RAM, song song |
| Độ phổ biến, thư viện | **Rất phổ biến**, hỗ trợ ở mọi nơi | Phổ biến, có sẵn trong Java | Ít hơn | Ngày càng tăng |
| Hỗ trợ Spring Security | `BCryptPasswordEncoder` | `Pbkdf2PasswordEncoder` | `SCryptPasswordEncoder` | `Argon2PasswordEncoder` |
| Hạn chế | Giới hạn 72 byte | Cần nhiều vòng lặp mới đủ an toàn | Cấu hình phức tạp hơn | Cần thư viện bổ sung (Bouncy Castle) |
| Khuyến nghị | **Đủ tốt cho hầu hết ứng dụng** | Khi cần chuẩn tuân thủ (FIPS) | Lựa chọn thay thế | **Lựa chọn hàng đầu cho hệ thống mới** |

**Gợi ý chọn:**
- **Ứng dụng thông thường bằng Spring Boot:** Bcrypt là lựa chọn an toàn và đơn giản nhất.
- **Hệ thống mới, yêu cầu bảo mật cao:** cân nhắc Argon2id.
- **Cần nâng cấp thuật toán sau này:** dùng `PasswordEncoderFactories.createDelegatingPasswordEncoder()`. Hash được lưu kèm tiền tố như `{bcrypt}...`, nên nhiều thuật toán cùng tồn tại và nâng cấp dần mà không bắt người dùng đổi mật khẩu.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

## Tổng kết

| Nguyên tắc | Ghi nhớ |
|---|---|
| Không lưu Plain Text | Lộ database là lộ tất cả tài khoản |
| Hashing, không phải encryption | Một chiều, chỉ cần so sánh khớp hay không |
| Không dùng MD5, SHA-1, SHA-256 | Quá nhanh, không có salt |
| Dùng Bcrypt (hoặc Argon2id) | Có salt tự động, có cost factor để làm chậm |
| Băm ở tầng ứng dụng | Dùng `encode` khi đăng ký, `matches` khi đăng nhập |
| Thông báo lỗi chung | "Email hoặc mật khẩu không đúng" |

**Cách nhớ nhanh:** Mật khẩu giống **chìa khóa nhà**, không ai cần giữ bản sao của bạn. Hệ thống chỉ giữ "dấu vân tay" (hash) của chìa khóa để kiểm tra khớp, và dấu vân tay đó phải **khó làm giả** (salt) và **mất thời gian để thử** (cost factor).
