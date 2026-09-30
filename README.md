# Bài tập thực hành JWT với Spring Boot 3 & Spring Security 6

Dự án hoàn thành theo bài giảng **JSON Web Token (WEBPR330479)** của **ThS. Nguyễn Hữu Trung - Khoa CNTT, Trường Đại học Sư Phạm Kỹ Thuật TP.HCM**.

---

## 1. Công nghệ sử dụng
- **Java**: 17
- **Spring Boot**: 3.3.4
- **Spring Security**: 6.x
- **Spring Data JPA & Hibernate**
- **Database**: MySQL (hoặc H2 in-memory khi test)
- **JSON Web Token**: **Nimbus JOSE + JWT** (`com.nimbusds:nimbus-jose-jwt` version `9.37.3`)
- **Thymeleaf & Bootstrap 5 & jQuery 3.7.1**
- **Lombok**

---

## 2. Cấu trúc dự án
```
jwt_springboot3
├── src/main/java/vn/iotstar
│   ├── configs
│   │   ├── ApplicationConfiguration.java   # Cấu hình UserDetailsService, PasswordEncoder, AuthenticationManager
│   │   ├── SecurityConfiguration.java      # Cấu hình SecurityFilterChain, CORS, Stateless session, JwtAuthenticationFilter
│   │   └── GlobalExceptionHandler.java     # Bắt và xử lý các ngoại lệ JWT, xác thực (BadCredentials, ExpiredJwt, SignatureException,...)
│   ├── controllers
│   │   ├── AuthenticationController.java   # API /auth/signup và /auth/login
│   │   ├── UserController.java             # API /users/me và /users/ (bảo vệ bởi JWT)
│   │   └── AuthController.java             # Điều hướng view Thymeleaf: /login và /user/profile
│   ├── entity
│   │   └── User.java                       # Entity User implements UserDetails
│   ├── filter
│   │   └── JwtAuthenticationFilter.java    # Filter trích xuất và xác thực Bearer JWT từ Authorization Header
│   ├── models
│   │   ├── LoginResponse.java              # Trả về token và expiresIn
│   │   ├── LoginUserModel.java             # Dữ liệu đăng nhập (email, password)
│   │   └── RegisterUserModel.java          # Dữ liệu đăng ký (email, password, fullName)
│   ├── repository
│   │   └── UserRepository.java             # JpaRepository tìm kiếm theo email
│   ├── services
│   │   ├── AuthenticationService.java      # Xử lý logic đăng ký, đăng nhập và xác thực
│   │   ├── JwtService.java                 # Xây dựng và kiểm tra tính hợp lệ của JWT token
│   │   └── UserService.java                # Lấy danh sách người dùng
│   └── JwtSpringboot3Application.java     # Class chạy ứng dụng Spring Boot
├── src/main/resources
│   ├── static
│   │   ├── images/                         # Thư mục chứa hình ảnh đại diện (u1.jpg,...)
│   │   └── js
│   │       └── mainjs.js                   # Xử lý Ajax đăng nhập, lưu localStorage.token, gọi API /users/me
│   ├── templates
│   │   ├── login.html                      # Giao diện đăng nhập với Ajax
│   │   └── profile.html                    # Giao diện thông tin tài khoản sau khi đăng nhập
│   └── application.properties              # Cấu hình port 8005, datasource MySQL, JWT secret key và expiration time
├── pom.xml
└── README.md
```

---

## 3. Cấu hình (`application.properties`)
```properties
spring.application.name=JWT_springboot3
server.port=8005

spring.datasource.url=jdbc:mysql://localhost:3306/jwt_springboot3?serverTimezone=UTC&allowPublicKeyRetrieval=true&useSSL=false
spring.datasource.username=root
spring.datasource.password=1234567@a$

## Hibernate properties
spring.jpa.hibernate.ddl-auto=update
spring.jpa.open-in-view=false

## JWT Config
security.jwt.secret-key=3cfa76ef14937c1c0ea519f8fc057a80fcd04a7420f8e8bcd0a7567c272e007b
# 1h in millisecond (3600000 ms)
security.jwt.expiration-time=3600000
```

---

## 4. Hướng dẫn kiểm thử API (Postman / cURL)

### Bước 1: Đăng ký tài khoản (`POST /auth/signup`)
- **URL**: `http://localhost:8005/auth/signup`
- **Method**: `POST`
- **Headers**: `Content-Type: application/json`
- **Body (JSON)**:
```json
{
    "email": "trungnh@hcmute.edu.vn",
    "password": "123456",
    "fullName": "Nguyễn Hữu Trung"
}
```
- **Response**: Trả về thông tin User đã được lưu vào database và mã hóa mật khẩu bằng BCrypt.

### Bước 2: Đăng nhập nhận JWT Token (`POST /auth/login`)
- **URL**: `http://localhost:8005/auth/login`
- **Method**: `POST`
- **Headers**: `Content-Type: application/json`
- **Body (JSON)**:
```json
{
    "email": "trungnh@hcmute.edu.vn",
    "password": "123456"
}
```
- **Response**:
```json
{
    "token": "eyJhbGciOiJIUzI1NiJ9...",
    "expiresIn": 3600000
}
```

### Bước 3: Gọi các API được bảo vệ bằng JWT Token
Thêm header:
`Authorization: Bearer <token_nhan_duoc>`

1. **Lấy thông tin người dùng đang đăng nhập**:
   - `GET http://localhost:8005/users/me`
2. **Lấy danh sách người dùng**:
   - `GET http://localhost:8005/users/`

---

## 5. Kiểm thử giao diện Web Ajax
1. Khởi chạy ứng dụng:
   ```bash
   ./mvnw spring-boot:run
   ```
2. Truy cập trình duyệt:
   - Trang đăng nhập: `http://localhost:8005/login`
   - Nhập email và password đã đăng ký, click **Login**.
   - Trình duyệt sẽ gửi Ajax request tới `/auth/login`, lưu `token` vào `localStorage.token`, sau đó tự động chuyển hướng tới `http://localhost:8005/user/profile`.
   - Tại trang `profile`, Ajax tự động đính kèm `Bearer <token>` vào request `/users/me` để lấy họ tên và ảnh hiển thị lên màn hình.
   - Nhấn **Logout** để xóa `localStorage.token` và quay về màn hình đăng nhập.

---

## 6. Xử lý ngoại lệ bảo mật (`GlobalExceptionHandler`)
- `BadCredentialsException`: HTTP 401 Unauthorized ("The username or password is incorrect")
- `AccountStatusException`: HTTP 403 Forbidden ("The account is locked")
- `AccessDeniedException`: HTTP 403 Forbidden ("You are not authorized to access this resource")
- `BadJOSEException / JOSEException / ParseException`: HTTP 401 Unauthorized ("The JWT signature is invalid")
- `BadJWTException`: HTTP 401 Unauthorized ("The JWT token has expired")
- Các lỗi khác: HTTP 500 Internal Server Error
