# Hướng dẫn từng bước: xây dựng hệ thống đặt vé bằng Spring Boot và PostgreSQL

> Tài liệu này biến ý tưởng dự án thành hướng dẫn có thể làm theo từ máy mới đến một MVP chạy được. Hãy hoàn thành từng chặng theo thứ tự; mỗi chặng có kết quả cần đạt để biết mình đã sẵn sàng đi tiếp.
>
> Ngày rà soát phiên bản: 23/09/2026. Chọn bản Spring Boot stable mới nhất trên Spring Initializr khi tạo dự án; không chọn bản SNAPSHOT. Java 21 là lựa chọn thực hành cho dự án này. Tài liệu Spring Boot hiện hành ghi Spring Boot 4.1.1 yêu cầu tối thiểu Java 17. Xem mục [Tài liệu chính thức](#tài-liệu-chính-thức) để kiểm tra lại khi bắt đầu.

---

## 1. Mục tiêu và phạm vi

Xây dựng API đặt vé có các chức năng:

1. Xem sự kiện và suất diễn.
2. Xem ghế trong một suất diễn.
3. Giữ nhiều ghế trong thời gian ngắn.
4. Tạo đơn hàng, mô phỏng thanh toán thành công hoặc thất bại.
5. Không cho hai người cùng giữ/đặt một ghế.
6. Tự nhả ghế khi thời gian giữ chỗ hết hạn.
7. Lưu schema bằng migration có phiên bản, không để Hibernate tự tạo bảng.
8. Chạy PostgreSQL bằng Docker Compose để mọi máy dev có cùng môi trường.

### MVP sẽ có

- Java 21, Spring Boot, Maven Wrapper.
- Spring Web, Spring Data JPA, Hibernate, Bean Validation, Actuator.
- PostgreSQL và Flyway.
- API REST, xử lý lỗi thống nhất.
- Khóa hàng PostgreSQL và ràng buộc unique để bảo vệ việc giữ ghế.
- Mock payment chạy trong ứng dụng.
- Docker Compose cho database.

### Để sau khi luồng chính chạy ổn

- Đăng nhập, JWT và phân quyền admin.
- Redis cho cache hoặc giữ chỗ phân tán nếu đã chứng minh cần thiết.
- RabbitMQ cho email/notification.
- Payment gateway thật.
- Prometheus/Grafana, deploy VPS, load test với k6.

**Quyết định kiến trúc:** PostgreSQL là nguồn dữ liệu chuẩn cho trạng thái giữ/đặt ghế. MVP không cần Redis lock. Giữ transaction DB ngắn và không gọi dịch vụ thanh toán bên ngoài khi transaction đang mở. Redis có thể bổ sung sau; nó không thay thế các ràng buộc và transaction của PostgreSQL.

---

## 2. Cài công cụ trên máy

Các lệnh trong tài liệu này viết cho **Linux Mint Terminal (Bash)**. Cần có JDK 21, Docker Engine + Docker Compose v2, IDE Java và Git.

### Xác định bản Linux Mint và nền tảng Ubuntu/Debian

Chạy:

~~~bash
cat /etc/os-release
grep -E '^(NAME|VERSION|VERSION_CODENAME|UBUNTU_CODENAME)=' /etc/os-release
dpkg --print-architecture
~~~

Linux Mint thường hiển thị codename riêng của Mint và codename của Ubuntu nền trong biến UBUNTU_CODENAME. Nếu đang dùng LMDE, máy dựa trên Debian và cần theo hướng dẫn Docker cho Debian. Khi cài từ repository Docker, dùng đúng Ubuntu/Debian nền; không dùng codename Mint làm codename Ubuntu.

### Cài Java 21, Git và công cụ cơ bản

Kiểm tra package Java có sẵn:

~~~bash
sudo apt update
apt-cache policy openjdk-21-jdk
~~~

Nếu có Candidate, cài:

~~~bash
sudo apt install openjdk-21-jdk git curl ca-certificates
~~~

Nếu không có Candidate, với Linux Mint dựa trên Ubuntu hãy mở **Software Sources**, bật repository **Universe** của Ubuntu nền rồi cập nhật package list. Với LMDE, kiểm tra các Debian repository đang bật và chạy lại apt update. Nếu package vẫn chưa có, cài Temurin JDK 21 theo hướng dẫn Linux chính thức của [Eclipse Adoptium](https://adoptium.net/installation/linux/); không hạ xuống Java cũ hơn chỉ để hoàn tất bước này.

### Cài Docker Engine và Compose

Nếu Docker đã có, chuyển thẳng sang bước kiểm tra. Nếu chưa, trước tiên xem package từ repository Mint/Ubuntu/Debian:

~~~bash
apt-cache policy docker.io docker-compose-v2
~~~

Nếu cả hai package có Candidate, cài Docker Engine và plugin Compose:

~~~bash
sudo apt install docker.io docker-compose-v2
sudo systemctl enable --now docker
~~~

Để dùng lệnh docker mà không thêm sudo, thêm tài khoản dev vào nhóm Docker rồi đăng xuất/đăng nhập lại:

~~~bash
sudo usermod -aG docker "$USER"
~~~

Nhóm Docker có quyền tương đương root trên máy; chỉ cấp cho tài khoản người dùng tin cậy trên máy dev.

Nếu các package đó không có Candidate, xác định nền ở bước trên rồi làm theo hướng dẫn chính thức cho [Ubuntu](https://docs.docker.com/engine/install/ubuntu/) hoặc [Debian](https://docs.docker.com/engine/install/debian/). Docker ghi rõ các bản Ubuntu phái sinh như Linux Mint không được kiểm thử/hỗ trợ chính thức; khi theo hướng dẫn Ubuntu cần thay bằng codename của Ubuntu nền, và cấu hình có thể cần điều chỉnh.

Kiểm tra:

~~~bash
java -version
javac -version
docker --version
docker compose version
docker run hello-world
git --version
~~~

Kết quả cần đạt: Java/Javac là phiên bản 21; Docker Engine đang chạy; docker compose chạy được; lệnh hello-world hoàn tất; Git có phiên bản. Nếu docker báo permission denied ngay sau khi thêm nhóm, đăng xuất Linux Mint rồi đăng nhập lại.

Không cần cài Maven toàn hệ thống: dự án chứa Maven Wrapper mvnw. Nếu có nhiều JDK, chọn JDK 21 trong IDE và kiểm tra terminal cũng đang dùng đúng Java.

---

## 3. Tạo project Spring Boot

1. Mở [Spring Initializr](https://start.spring.io/).
2. Chọn các giá trị:
   - Project: **Maven**
   - Language: **Java**
   - Spring Boot: bản **stable mới nhất**, không chọn SNAPSHOT
   - Group: **vn.datve**
   - Artifact: **dat-ve**
   - Name: **dat-ve**
   - Package name: **vn.datve**
   - Packaging: **Jar**
   - Java: **21**
3. Thêm dependencies:
   - **Spring Web** (tạo REST API)
   - **Spring Data JPA** (làm việc với database qua JPA)
   - **Validation** (kiểm tra dữ liệu request)
   - **PostgreSQL Driver**
   - **Flyway Migration**
   - **Spring Boot Actuator** (health check)
4. Chọn **Generate**, giải nén file tải về vào workspace, rồi mở thư mục chứa pom.xml trong IDE.
5. Chờ IDE tải dependencies. Chạy ứng dụng lần đầu để xác nhận project khởi tạo được:

~~~bash
./mvnw spring-boot:run
~~~

Nếu Linux Mint báo Permission denied khi chạy Maven Wrapper, cấp quyền thực thi một lần:

~~~bash
chmod +x mvnw
~~~

Sau đó chạy lại ./mvnw spring-boot:run.

Nếu ứng dụng báo không có database, đó là dự kiến ở bước này vì PostgreSQL chưa được chạy. Mục tiêu của lần chạy đầu là xác nhận Maven Wrapper tải được dependency và mã nguồn biên dịch.

### Kiểm tra dependency Flyway

Flyway cần các thư viện phù hợp với phiên bản Spring Boot đã chọn. Kiểm tra pom.xml sau khi tải project. Nếu Initializr chưa thêm module database cho Flyway, thêm dependency org.flywaydb:flyway-database-postgresql cùng phiên bản do Spring Boot quản lý. Không tự đặt version riêng nếu chưa có lý do.

---

## 4. Tạo cấu trúc thư mục ban đầu

Trong src/main/java/vn/datve, tạo package theo chức năng:

~~~text
vn.datve
├── DatVeApplication.java
├── common
│   ├── exception
│   └── response
├── config
├── event
│   ├── EventController.java
│   ├── EventService.java
│   ├── EventRepository.java
│   ├── Event.java
│   └── dto
├── showtime
├── seat
├── booking
├── payment
└── user
~~~

Trong src/main/resources:

~~~text
src/main/resources
├── application.yml
└── db
    └── migration
        └── V1__init_schema.sql
~~~

Quy ước quan trọng:

- Controller: nhận HTTP request, kiểm tra đầu vào cơ bản và gọi service.
- Service: chứa quy tắc nghiệp vụ và ranh giới transaction.
- Repository: truy vấn database.
- Entity: ánh xạ bảng.
- dto: request/response API. Không trả thẳng JPA entity ra ngoài.
- db/migration: các file SQL phiên bản hóa do Flyway quản lý.
- Không tạo package repository/service quá lớn chứa mọi chức năng; gom theo miền nghiệp vụ như booking, event.

---

## 5. Khởi chạy PostgreSQL bằng Docker Compose

Tại thư mục gốc của dự án, tạo file .env.example:

~~~dotenv
POSTGRES_DB=datve_db
POSTGRES_USER=datve_user
POSTGRES_PASSWORD=local_dev_only_12345
~~~

Tạo .env từ file mẫu:

~~~bash
cp -n .env.example .env
~~~

.env là cấu hình dùng riêng trên máy dev; không commit file này. Tùy chọn -n giữ nguyên .env nếu bạn đã có file cấu hình riêng. Được phép commit .env.example vì nó không chứa mật khẩu cá nhân hay production secret.

Tạo compose.yaml tại thư mục gốc:

~~~yaml
services:
  postgres:
    image: postgres:18
    container_name: datve-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-datve_db}
      POSTGRES_USER: ${POSTGRES_USER:-datve_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-local_dev_only_12345}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  postgres_data:
~~~

Chạy PostgreSQL:

~~~bash
docker compose up -d postgres
docker compose ps
docker compose logs -f postgres
~~~

Đợi đến khi container healthy hoặc log báo database sẵn sàng nhận kết nối. Dừng xem log bằng Ctrl+C; việc đó không dừng container.

### Kiểm tra database trực tiếp

Mở terminal khác:

~~~bash
docker compose exec postgres psql -U datve_user -d datve_db
~~~

Trong psql, chạy:

~~~sql
SELECT current_database(), current_user, version();
\conninfo
\q
~~~

Kết quả phải cho thấy database datve_db, user datve_user, và kết nối đến PostgreSQL trong container.

### Dữ liệu local được lưu ở đâu?

Named volume postgres_data giữ dữ liệu sau khi container dừng hoặc được tạo lại. Lệnh sau xóa container nhưng giữ volume:

~~~bash
docker compose down
~~~

Lệnh sau xóa cả dữ liệu database local:

~~~bash
docker compose down -v
~~~

Chỉ chạy down -v khi chủ động muốn tạo database sạch để làm lại từ đầu.

---

## 6. Cấu hình Spring kết nối PostgreSQL

Tạo src/main/resources/application.yml:

~~~yaml
spring:
  application:
    name: dat-ve
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/datve_db}
    username: ${DB_USERNAME:datve_user}
    password: ${DB_PASSWORD:local_dev_only_12345}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
  flyway:
    enabled: true
    locations: classpath:db/migration

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: never

server:
  port: 8080
~~~

Spring tự đọc biến môi trường trong cấu hình trên. Với cài đặt mặc định, ứng dụng chạy trực tiếp trên máy kết nối đến localhost:5432.

**Không bật spring.jpa.hibernate.ddl-auto: update để tạo schema.** Flyway là chủ sở hữu schema; Hibernate validate chỉ kiểm tra entity có khớp với schema đã migrate hay không. Nhờ vậy, thay đổi bảng luôn được lưu thành migration có thể review.

### Biến môi trường khi chạy trên máy dev

Các giá trị mặc định ở trên khớp với .env.example. Nếu bạn đã đổi thông tin database trong .env, hãy đưa các giá trị tương ứng vào terminal trước khi chạy Spring. Compose dùng .env để khởi tạo container; Spring Boot không tự đọc file .env.

Bash trên Linux Mint:

~~~bash
export DB_URL='jdbc:postgresql://localhost:5432/datve_db'
export DB_USERNAME='datve_user'
export DB_PASSWORD='local_dev_only_12345'
./mvnw spring-boot:run
~~~

Nếu chưa đổi file .env.example, có thể dùng cấu hình mặc định trong application.yml và chỉ chạy:

~~~bash
./mvnw spring-boot:run
~~~

---

## 7. Tạo migration đầu tiên và xác nhận kết nối

Tạo file src/main/resources/db/migration/V1__init_schema.sql:

~~~sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'USER',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT users_role_check CHECK (role IN ('USER', 'ADMIN'))
);

CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    venue VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE showtimes (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(id),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT showtimes_valid_time_check CHECK (end_time > start_time),
    CONSTRAINT showtimes_no_overlap
        EXCLUDE USING gist (
            event_id WITH =,
            tstzrange(start_time, end_time, '[)') WITH &&
        )
);

CREATE TABLE seats (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    showtime_id BIGINT NOT NULL REFERENCES showtimes(id),
    seat_code VARCHAR(12) NOT NULL,
    category VARCHAR(30) NOT NULL DEFAULT 'STANDARD',
    face_value NUMERIC(12, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT seats_nonnegative_price_check CHECK (face_value >= 0),
    CONSTRAINT seats_showtime_code_unique UNIQUE (showtime_id, seat_code),
    CONSTRAINT seats_showtime_id_unique UNIQUE (showtime_id, id)
);

CREATE TABLE bookings (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    showtime_id BIGINT NOT NULL REFERENCES showtimes(id),
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    total_amount NUMERIC(12, 2) NOT NULL,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT bookings_status_check
        CHECK (status IN ('PENDING', 'PAYMENT_PROCESSING', 'CONFIRMED', 'CANCELLED', 'EXPIRED')),
    CONSTRAINT bookings_id_showtime_unique UNIQUE (id, showtime_id),
    CONSTRAINT bookings_nonnegative_amount_check CHECK (total_amount >= 0)
);

CREATE INDEX bookings_user_created_idx ON bookings (user_id, created_at DESC);
CREATE INDEX bookings_status_expiry_idx ON bookings (status, expires_at);

CREATE TABLE booking_seats (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_id BIGINT NOT NULL,
    showtime_id BIGINT NOT NULL,
    seat_id BIGINT NOT NULL,
    reservation_status VARCHAR(20) NOT NULL,
    unit_price NUMERIC(12, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT booking_seats_status_check
        CHECK (reservation_status IN ('HELD', 'BOOKED', 'RELEASED')),
    CONSTRAINT booking_seats_nonnegative_price_check CHECK (unit_price >= 0),
    CONSTRAINT booking_seats_booking_seat_unique UNIQUE (booking_id, seat_id),
    CONSTRAINT booking_seats_booking_same_showtime_fk
        FOREIGN KEY (booking_id, showtime_id) REFERENCES bookings (id, showtime_id),
    CONSTRAINT booking_seats_seat_same_showtime_fk
        FOREIGN KEY (showtime_id, seat_id) REFERENCES seats (showtime_id, id)
);

CREATE INDEX booking_seats_booking_idx ON booking_seats (booking_id);
CREATE INDEX booking_seats_showtime_seat_idx ON booking_seats (showtime_id, seat_id);

CREATE UNIQUE INDEX booking_seats_one_active_reservation_per_seat
    ON booking_seats (seat_id)
    WHERE reservation_status IN ('HELD', 'BOOKED');

CREATE TABLE payments (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_id BIGINT NOT NULL REFERENCES bookings(id),
    idempotency_key VARCHAR(100) NOT NULL UNIQUE,
    provider VARCHAR(50) NOT NULL DEFAULT 'MOCK',
    provider_reference VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    amount NUMERIC(12, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT payments_status_check
        CHECK (status IN ('PENDING', 'PROCESSING', 'SUCCESS', 'FAILED')),
    CONSTRAINT payments_nonnegative_amount_check CHECK (amount >= 0)
);

CREATE INDEX payments_booking_idx ON payments (booking_id);
~~~

### Mô hình trạng thái của ghế

seats là danh sách ghế có thật của một suất diễn; booking_seats ghi lịch sử ghế được giữ hoặc đặt:

- HELD: đang nằm trong thời hạn giữ ghế.
- BOOKED: thanh toán đã thành công.
- RELEASED: booking bị hủy, thanh toán thất bại hoặc thời hạn giữ chỗ hết.

Partial unique index chỉ cho phép tối đa một dòng HELD hoặc BOOKED cho mỗi ghế. Khi ghế được nhả, trạng thái chuyển sang RELEASED; vì vậy ghế có thể được đặt lại và lịch sử booking vẫn còn.

Khi xử lý yêu cầu mới, ứng dụng phải phát hiện và chuyển các hold hết hạn sang RELEASED trước khi thêm hold mới. Chỉ dựa vào thời gian trôi qua là chưa đủ vì index không tự kiểm tra expires_at.

### Chạy Flyway và kiểm tra

Khởi động app khi PostgreSQL đang chạy:

~~~bash
./mvnw spring-boot:run
~~~

Theo dõi log. Cần thấy Flyway migrate thành công và Spring khởi động mà không có lỗi datasource, migration hoặc schema validation.

Mở:

- [http://localhost:8080/actuator/health](http://localhost:8080/actuator/health)

Kết quả mong đợi là trạng thái UP. Không mở các Actuator endpoint nhạy cảm ra Internet.

Sau đó kiểm tra các bảng:

~~~bash
docker compose exec postgres psql -U datve_user -d datve_db
~~~

~~~sql
\dt
SELECT installed_rank, version, description, success
FROM flyway_schema_history
ORDER BY installed_rank;
\q
~~~

Phải thấy các bảng trong migration và bản V1 có success = true. Từ đây đã xác nhận được cả ba phần: ứng dụng chạy, Spring kết nối PostgreSQL và Flyway tạo schema.

### Quy tắc khi schema thay đổi

- Migration đã chạy ở máy khác hoặc môi trường dùng chung thì không sửa nội dung file cũ.
- Thêm file mới, ví dụ V2__add_event_status.sql, rồi để Flyway áp dụng.
- Đặt tên migration dạng V<number>__mô_tả.sql, tăng số phiên bản liên tục.
- Không vừa để Hibernate update vừa dùng Flyway.
- Nếu migration mới lỗi ở local, đọc lỗi và sửa migration mới trước khi đã chia sẻ migration đó. Với dữ liệu local bỏ được, có thể xóa volume rồi dựng lại; không làm vậy với database có dữ liệu cần giữ.

---

## 8. Tạo dữ liệu phát triển để xem API

Sau khi đã chạy V1, có thể tạo dữ liệu dev trực tiếp để bắt đầu viết API:

~~~sql
INSERT INTO events (name, description, venue)
VALUES ('Đêm nhạc mùa thu', 'Sự kiện mẫu cho môi trường phát triển', 'Nhà hát trung tâm')
RETURNING id;
~~~

Dùng event_id trả về ở câu lệnh trên:

~~~sql
INSERT INTO showtimes (event_id, start_time, end_time)
VALUES (1, now() + interval '7 days', now() + interval '7 days 2 hours')
RETURNING id;
~~~

Dùng showtime_id trả về để tạo 20 ghế:

~~~sql
INSERT INTO seats (showtime_id, seat_code, category, face_value)
SELECT 1,
       chr(65 + ((n - 1) / 10)) || (((n - 1) % 10) + 1)::text,
       'STANDARD',
       150000
FROM generate_series(1, 20) AS n;
~~~

Các số 1 ở ví dụ chỉ đúng khi ID trả về của bạn là 1. Thay bằng ID thực tế. Để tạo dữ liệu có thể dùng lại nhiều lần trên mọi môi trường dev, sau này chuyển seed này thành script dev riêng; không đặt mật khẩu mặc định hoặc tài khoản admin yếu vào migration production.

Kiểm tra:

~~~sql
SELECT s.id, s.seat_code, s.category, s.face_value
FROM seats s
WHERE s.showtime_id = 1
ORDER BY s.seat_code;
~~~

---

## 9. Viết ứng dụng theo từng lát chức năng

Không viết toàn bộ entity và endpoint cùng lúc. Mỗi lát nên đi từ database đến API và xác nhận được kết quả.

### Lát 1: Event

1. Tạo Event entity tương ứng bảng events.
2. Tạo EventRepository extends JpaRepository<Event, Long>.
3. Tạo response DTO chỉ gồm trường API cần trả.
4. Tạo EventService để gọi repository.
5. Tạo GET /api/events và GET /api/events/{id}.
6. Trả 404 Not Found khi không có event.
7. Kiểm tra JSON trả về không chứa trường nội bộ hoặc mật khẩu.

### Lát 2: Showtime và ghế

1. Tạo Showtime và Seat entity.
2. Tạo endpoint:
   - GET /api/events/{eventId}/showtimes
   - GET /api/showtimes/{showtimeId}/seats
3. Với danh sách ghế, trả seatCode, category, faceValue và trạng thái hiện tại.
4. Trạng thái hiện tại lấy từ booking_seats:
   - có BOOKED → BOOKED;
   - có hold chưa hết hạn → HELD;
   - không có reservation đang hiệu lực → AVAILABLE.
5. Dùng pagination cho danh sách event nếu có thể tăng lớn; không trả toàn bộ dữ liệu không giới hạn.

### Lát 3: Chuẩn hóa lỗi API

Tạo exception handler toàn cục bằng @RestControllerAdvice. Thống nhất JSON lỗi, ví dụ:

~~~json
{
  "code": "SEAT_UNAVAILABLE",
  "message": "Một hoặc nhiều ghế không còn trống.",
  "timestamp": "2026-09-23T10:00:00Z"
}
~~~

Mã HTTP gợi ý:

| Trường hợp | HTTP |
|---|---:|
| Request sai định dạng/thiếu trường | 400 |
| Chưa đăng nhập hoặc token sai | 401 |
| Không có quyền | 403 |
| Không tìm thấy event, suất diễn, booking | 404 |
| Ghế vừa được người khác giữ/đặt | 409 |
| Lỗi ngoài dự kiến | 500 |

Không trả stack trace, SQL, password hoặc secret trong response.

---

## 10. Luồng giữ ghế an toàn khi có request đồng thời

Endpoint đề xuất:

- POST /api/showtimes/{showtimeId}/bookings
- Request: danh sách ghế muốn giữ.
- Response: booking ID, danh sách ghế, tổng tiền, thời điểm hết hạn.
- Thành công trả HTTP 201 Created; request cạnh tranh thua trả 409 Conflict với code SEAT_UNAVAILABLE. Hai status này là contract để test k6 ở mục 15.

Ví dụ request:

~~~json
{
  "seatIds": [11, 12]
}
~~~

Service giữ ghế phải chạy trong **một transaction ngắn**:

1. Kiểm tra request không rỗng, không có seat ID trùng, số ghế dưới giới hạn nghiệp vụ.
2. Lock các hàng trong seats tương ứng suất diễn theo thứ tự tăng dần ID, dùng SELECT ... FOR UPDATE.
3. Xác nhận đã tìm thấy đủ tất cả ghế và tất cả thuộc đúng suất diễn. Nếu thiếu ghế, dừng với 404 hoặc 400.
4. Tìm booking PENDING có hold đã hết hạn chạm vào những ghế này. Chuyển toàn bộ booking_seats của các booking hết hạn sang RELEASED, rồi chuyển booking tương ứng sang EXPIRED.
5. Kiểm tra ghế còn trạng thái AVAILABLE; nếu bất kỳ ghế nào đã HELD hoặc BOOKED, từ chối toàn bộ yêu cầu bằng 409. Không giữ một phần danh sách ghế.
6. Tính giá ở server từ bảng seats; không tin giá do client gửi.
7. Tạo booking PENDING, đặt expires_at (ví dụ thời điểm hiện tại cộng 5 phút), rồi thêm các dòng booking_seats trạng thái HELD.
8. Commit transaction.
9. Nếu unique index báo xung đột, chuyển lỗi đó thành 409 SEAT_UNAVAILABLE; không gửi SQL error cho client.

Lock các hàng ghế theo cùng một thứ tự giảm khả năng deadlock khi hai người chọn nhiều ghế giống nhau. Unique index là lớp bảo vệ cuối cùng nếu có code path nào quên kiểm tra trạng thái.

### Vì sao không dùng SKIP LOCKED cho danh sách ghế khách đã chọn?

Khách hàng yêu cầu đúng những ghế cụ thể. Nếu DB bỏ qua ghế đang bị lock mà ứng dụng vẫn giữ phần còn lại, API có thể tạo booking thiếu ghế. Với đặt ghế, lock đủ ghế đã yêu cầu rồi thành công toàn bộ hoặc trả xung đột. SKIP LOCKED phù hợp hơn cho worker xử lý một hàng đợi công việc theo lô.

### Transaction và lỗi unique

Nếu unique partial index phát sinh lỗi khi insert, PostgreSQL đánh dấu transaction hiện tại là lỗi. Không bắt exception đó rồi tiếp tục chạy query trong chính transaction đó. Hãy để transaction rollback, bắt lỗi ở ranh giới service/controller thích hợp và chuyển thành HTTP 409.

### Khi một booking giữ nhiều ghế

Nếu một ghế đã hết hạn, giải phóng tất cả ghế còn lại trong cùng booking để người mua không phải thanh toán một đơn bị thiếu ghế. Việc cập nhật booking và toàn bộ dòng booking_seats phải ở trong cùng transaction.

---

## 11. Hủy booking và tự giải phóng hold hết hạn

### Hủy booking

POST /api/bookings/{bookingId}/cancel:

1. Tìm booking và kiểm tra người gọi có quyền hủy.
2. Lock booking và các ghế liên quan theo một thứ tự thống nhất.
3. Chỉ cho hủy booking PENDING (hoặc quy định nghiệp vụ cho phép thêm trạng thái khác).
4. Chuyển booking thành CANCELLED.
5. Chuyển các dòng booking_seats HELD thành RELEASED.
6. Commit transaction. Nếu booking đã CONFIRMED, cần chính sách hoàn tiền riêng; không âm thầm đặt lại ghế.

### Worker hết hạn

Dùng Spring Scheduler để quét booking đến hạn:

~~~text
PENDING + expires_at <= thời gian hiện tại
    -> booking EXPIRED
    -> booking_seats HELD chuyển thành RELEASED
~~~

- Cập nhật booking và ghế trong một transaction.
- Dùng batch nhỏ.
- Có index (status, expires_at) như migration đã tạo.
- Nếu chạy nhiều instance ứng dụng, worker phải phối hợp bằng lock DB; có thể dùng FOR UPDATE SKIP LOCKED để mỗi worker lấy các booking khác nhau.
- Worker chỉ là cách dọn dẹp chủ động. API giữ ghế vẫn phải tự xử lý hold đã hết hạn cho các ghế khách đang chọn.
- Dùng UTC cho dữ liệu; format thời gian có timezone ở API.

---

## 12. Mock payment không giữ transaction trong lúc chờ

Endpoint gợi ý:

- POST /api/bookings/{bookingId}/payments
- Client gửi Idempotency-Key riêng cho lần yêu cầu.
- Khi phát triển, mock nhận kết quả mô phỏng SUCCESS hoặc FAIL; không bật khả năng chọn kết quả thanh toán cho user thật.

Cấu trúc:

~~~text
PaymentProvider
└── MockPaymentProvider
~~~

Quy trình an toàn:

1. **Transaction ngắn A:** lock booking, xác thực chủ sở hữu, xác nhận còn hạn và đang PENDING. Tạo payment PROCESSING, chuyển booking sang PAYMENT_PROCESSING, commit.
2. Gọi PaymentProvider **ngoài transaction**. Mock có thể trả kết quả tức thời; nếu mô phỏng độ trễ thì không giữ DB lock trong lúc chờ.
3. **Transaction ngắn B:** lock booking/payment, xác nhận payment chưa được xử lý trước đó, rồi:
   - Thành công: payment SUCCESS, booking CONFIRMED, các ghế HELD thành BOOKED.
   - Thất bại: payment FAILED, booking CANCELLED, các ghế HELD thành RELEASED.
4. Request lặp lại với cùng Idempotency-Key trả lại kết quả đã lưu thay vì tạo payment trùng.
5. Chặn worker hết hạn xử lý booking đang PAYMENT_PROCESSING; đồng thời đặt timeout/reconciliation cho payment bị treo để không giữ ghế vô hạn.

Mock không có kết nối với ngân hàng và không chứng minh được hệ thống đã thanh toán tiền thật. Nó chỉ giúp kiểm thử trạng thái đơn, idempotency và việc giữ/nhả ghế. Khi tích hợp gateway, xử lý callback phải xác minh chữ ký callback và chống xử lý lặp.

**Không** đặt Thread.sleep hoặc lời gọi HTTP gateway bên trong method @Transactional. Việc đó kéo dài row lock, giữ connection trong pool và làm nghẽn các request khác.

---

## 13. Đăng nhập và quyền admin

Chỉ thêm sau khi event/showtime/seat/booking hoạt động tốt.

1. Tạo endpoint POST /api/auth/register và POST /api/auth/login.
2. Mã hóa mật khẩu bằng PasswordEncoder phù hợp; không lưu mật khẩu dạng rõ.
3. Bảo vệ endpoint tạo event/showtime bằng role ADMIN.
4. Booking và payment chỉ cho chủ booking thao tác; không dùng ID đoán được để bỏ qua kiểm tra quyền.
5. Nếu dùng JWT, quản lý secret qua secret manager hoặc biến môi trường trên deploy; không commit secret.
6. Giới hạn Actuator chỉ ở health/info cần thiết. Không expose env, config props hoặc dump heap công khai.

API mở đầu:

| Method | Endpoint | Mục đích |
|---|---|---|
| POST | /api/auth/register | Tạo tài khoản USER |
| POST | /api/auth/login | Xác thực và trả thông tin phiên |
| GET | /api/events | Danh sách sự kiện |
| GET | /api/events/{id} | Chi tiết sự kiện |
| POST | /api/events | ADMIN tạo sự kiện |
| POST | /api/showtimes | ADMIN tạo suất diễn và sơ đồ ghế |
| GET | /api/showtimes/{id}/seats | Xem trạng thái ghế |
| POST | /api/showtimes/{id}/bookings | Giữ ghế và tạo booking |
| GET | /api/bookings/{id} | Xem booking của chủ sở hữu |
| GET | /api/bookings/me | Lịch sử booking của tài khoản |
| POST | /api/bookings/{id}/cancel | Hủy booking còn hiệu lực |
| POST | /api/bookings/{id}/payments | Chạy mock payment |

---

## 14. HikariCP và cấu hình kết nối

Spring Boot dùng HikariCP làm connection pool mặc định khi có JDBC. Chưa tune pool ở giai đoạn đầu.

Khi đến chặng đo pool, cấu hình tối thiểu như sau trong application.yml để thay maximum pool size mà không sửa source code:

~~~yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      minimum-idle: 2
      connection-timeout: 3000
~~~

Sau khi API hoạt động, chạy cùng một kịch bản đọc với pool 5 và pool 10:

1. Xem log và metrics connection pool.
2. Đo số connection DB tối đa, request đồng thời, thời gian chờ connection.
3. Chỉ đổi pool size; giữ nguyên máy, dữ liệu, git commit và workload.
4. Warm-up 1 phút, đo 3 phút, lặp 3 lần cho từng pool.
5. Ghi lại RPS thực tế, p95/p99, dropped iterations, 5xx và số connection DB lớn nhất.
6. Chọn cấu hình theo số đo và giới hạn PostgreSQL; không đặt pool size lớn chỉ vì số lượng virtual user lớn.

Pool của mọi instance cộng lại phải nằm trong giới hạn connection thực tế của database. Mỗi request không được giữ transaction mở lâu hơn mức cần thiết.

---

## 15. Kiểm tra nghiệp vụ và đo tải

### 15.1 Vấn đề dự án giải quyết và chỉ số chính

**Vấn đề:** tại thời điểm có nhiều người cùng chọn ghế cuối, hai request có thể cùng đọc trạng thái AVAILABLE rồi cùng tạo booking. Khách bị bán trùng ghế, đội hỗ trợ phải hoàn tiền/đổi chỗ và trải nghiệm mua vé giảm.

**Chỉ số chính của dự án:** số ghế có nhiều booking đang hiệu lực sau một đợt cạnh tranh. Mục tiêu là **0 ghế bị giữ/đặt trùng**. Chỉ số này gắn trực tiếp với tác dụng của hệ thống; RPS cao không có ý nghĩa nếu bán trùng ghế.

**Mục tiêu kiểm thử được đặt trước, chưa phải kết quả đã đo:**

- Một ghế, 100 request cạnh tranh trong một lượt: đúng **1** response tạo booking (201), đúng **99** response xung đột nghiệp vụ (409), **0** response 5xx, **0** ghế active bị trùng.
- Lặp với **10 ghế thử nghiệm riêng biệt**, mỗi ghế chỉ dùng một lượt: tổng **1.000** request, **10** booking thắng, **990** xung đột, **0** lỗi 5xx, **0** ghế active trùng.
- Lượt đọc danh sách ghế ở tải đã định: mục tiêu khởi điểm là p95 dưới **300 ms** ở **25 request/giây** trên máy ghi trong biên bản. Đây là mục tiêu dev để kiểm tra trải nghiệm đọc, không phải lời hứa hiệu năng production.

Nếu phần cứng không đạt mục tiêu p95, trước tiên ghi kết quả thật và tìm nút thắt; không đổi số đo thành số đẹp. Chỉ ghi một chỉ số CV là “đã tối ưu” khi có số đo trước/sau, workload và môi trường có thể giải thích.

**Phép đo trước/sau để kể câu chuyện kỹ thuật rõ hơn:** trong test-only baseline, 50 transaction riêng cùng kiểm tra một ghế là trống; dùng barrier trong test để cả 50 transaction đọc xong trước khi bất kỳ transaction nào insert. Bản baseline không dùng row lock và chạy trên schema benchmark bỏ partial unique index, nên tạo ra 50 hold cho một ghế, tức **49 hold vượt mức**. Chạy cùng 50 caller qua implementation thật có row lock và unique index: kỳ vọng 1 hold, 49 conflict và **0 hold vượt mức**. Lặp trên 10 ghế thử riêng cho baseline 490 hold vượt mức và implementation thật 0. Đây là thí nghiệm race condition có chủ đích để chứng minh invariant; **không phải tỷ lệ bán trùng của production**. Chỉ dùng kết quả sau khi test thật chạy xong.

### 15.2 Các lớp kiểm tra và công cụ

1. **Unit test:** kiểm tra validation và chuyển trạng thái booking/payment; không cần khởi động database.
2. **Integration test:** chạy với PostgreSQL thật bằng Testcontainers. Dùng lớp này để kiểm tra Flyway, partial unique index, khóa hàng và exclusion constraint. H2 không thay được PostgreSQL cho các hành vi riêng này.
3. **Concurrency test xác định:** chạy nhiều thread cùng chờ tại một barrier rồi gọi service trên các transaction độc lập. Dùng **50 thread × 10 lượt**, mỗi lượt một ghế mới; kỳ vọng mỗi lượt có 1 thành công và 49 xung đột.
4. **End-to-end/load test:** dùng k6 gọi ứng dụng đang chạy qua HTTP. Bài này đo hành vi của cả API, connection pool và PostgreSQL, khác với unit/integration test.

Trong integration test, có thể giữ baseline check-then-insert ở test helper riêng và schema benchmark dùng một lần. Không thêm baseline dễ lỗi thành endpoint, profile production hay nhánh runtime trong ứng dụng. Đặt barrier sau bước đọc AVAILABLE để phép đo trước/sau thực sự tạo cùng một interleaving; implementation production chạy nguyên vẹn với lock/index.

Kết quả kỳ vọng của phép đo có kiểm soát:

| Cách xử lý | Ghế thử | Transaction/ghế | Tổng lần thử | Hold được tạo | Conflict | Hold vượt mức |
|---|---:|---:|---:|---:|---:|---:|
| Baseline test-only, không lock/index | 10 | 50 | 500 | 500 | 0 | 490 |
| Implementation thật, row lock + index | 10 | 50 | 500 | 10 | 490 | 0 |

Đây là số **kỳ vọng trong test barrier**, chưa phải kết quả dự án. Nó cô lập đúng race “đọc trống rồi cùng ghi”; k6 ở mục sau kiểm tra API HTTP thực tế.

Sau khi luồng nghiệp vụ đã có, thêm Testcontainers PostgreSQL dependency ở test scope. Chạy kiểm tra bằng Maven Wrapper:

~~~bash
./mvnw clean verify
~~~

### 15.3 Kiểm tra nghiệp vụ bắt buộc

| ID | Chuẩn bị và thao tác | Kết quả phải đúng |
|---|---|---|
| B01 | Một user giữ 1 ghế còn trống | 201; booking PENDING; đúng 1 dòng HELD; giá lấy từ DB |
| B02 | Gửi cùng seat ID hai lần trong một request | 400; không tạo booking |
| B03 | Chọn 1 ghế trống và 1 ghế đang giữ | 409; transaction rollback; ghế trống không bị giữ một phần |
| B04 | 100 request đồng thời giữ một ghế | 1 response 201, 99 response 409, không có 5xx, chỉ 1 booking/hold active |
| B05 | Hủy booking PENDING | booking CANCELLED; ghế RELEASED; user khác giữ lại được |
| B06 | Đặt expires_at về quá khứ rồi chạy expiry worker | booking EXPIRED; toàn bộ ghế của đơn RELEASED; giữ lại được |
| B07 | Mock payment thành công | payment SUCCESS; booking CONFIRMED; ghế BOOKED |
| B08 | Mock payment thất bại | payment FAILED; booking đóng theo state machine; ghế RELEASED |
| B09 | Gửi cùng Idempotency-Key 10 lần | chỉ một payment record; mọi lần trả cùng kết quả đã lưu |
| B10 | Tạo hai suất cùng event có khoảng giờ giao nhau | insert thứ hai bị PostgreSQL từ chối |
| B11 | User A đọc/hủy booking của user B | 404 hoặc 403 theo quy ước API; không lộ dữ liệu |
| B12 | Gửi JSON sai hoặc thiếu seat ID | 400; không ghi dữ liệu một phần |

Sau mỗi test quan trọng, kiểm tra cả HTTP response lẫn DB. Có thể kiểm tra không còn ghế active trùng bằng truy vấn:

~~~sql
SELECT seat_id, count(*) AS active_reservations
FROM booking_seats
WHERE reservation_status IN ('HELD', 'BOOKED')
GROUP BY seat_id
HAVING count(*) > 1;
~~~

Kết quả phải là **0 dòng**. Unique index khiến database từ chối trường hợp trùng; truy vấn này vẫn hữu ích để ghi lại invariant sau một đợt kiểm thử.

### 15.4 Test đồng thời bằng k6

Tạo file scripts/k6/seat-contention.js sau khi endpoint giữ ghế chạy được. Script dưới đây giả định response tạo đơn là 201, xung đột ghế là 409 với JSON code SEAT_UNAVAILABLE. Nếu API chọn tên code khác, đổi assertion cho khớp hợp đồng API.

~~~javascript
import http from 'k6/http';
import { check } from 'k6';
import { Counter } from 'k6/metrics';

const created = new Counter('hold_created');
const conflicts = new Counter('hold_conflict');
const unexpected = new Counter('unexpected_response');

const baseUrl = __ENV.BASE_URL || 'http://localhost:8080';
const showtimeId = __ENV.SHOWTIME_ID;
const seatId = Number(__ENV.SEAT_ID);
const bearerToken = __ENV.BEARER_TOKEN;
const expectedBookingStatuses = http.expectedStatuses(201, 409);

export const options = {
  scenarios: {
    one_seat_contention: {
      executor: 'per-vu-iterations',
      vus: 100,
      iterations: 1,
      maxDuration: '1m',
    },
  },
  thresholds: {
    hold_created: ['count==1'],
    hold_conflict: ['count==99'],
    unexpected_response: ['count==0'],
    checks: ['rate==1'],
    http_req_failed: ['rate==0'],
    http_req_duration: ['p(95)<1000'],
  },
};

export default function () {
  const headers = { 'Content-Type': 'application/json' };
  if (bearerToken) {
    headers.Authorization = 'Bearer ' + bearerToken;
  }

  const response = http.post(
    baseUrl + '/api/showtimes/' + showtimeId + '/bookings',
    JSON.stringify({ seatIds: [seatId] }),
    {
      headers,
      tags: { endpoint: 'hold', test: 'one-seat-contention' },
      responseCallback: expectedBookingStatuses,
    },
  );

  let isExpectedBusinessResult = false;
  if (response.status === 201) {
    created.add(1);
    isExpectedBusinessResult = true;
  } else if (response.status === 409) {
    let errorCode = '';
    try {
      errorCode = response.json('code');
    } catch (_) {
      errorCode = '';
    }
    if (errorCode === 'SEAT_UNAVAILABLE') {
      conflicts.add(1);
      isExpectedBusinessResult = true;
    } else {
      unexpected.add(1);
    }
  } else {
    unexpected.add(1);
  }

  check(response, {
    'response is the expected booking outcome': () => isExpectedBusinessResult,
  });
}
~~~

Trước mỗi lượt:

1. Dùng một suất diễn chỉ dành cho benchmark và một ghế chưa từng dùng.
2. Cấp JWT của tài khoản test nếu endpoint đã bật authentication.
3. Không chạy expiry worker hoặc test khác trên cùng fixture trong lúc k6 chạy.
4. Lưu SHOWTIME_ID, SEAT_ID, mã commit và output k6.

Chạy một lượt:

~~~bash
k6 run \
  -e BASE_URL=http://localhost:8080 \
  -e SHOWTIME_ID=9001 \
  -e SEAT_ID=901001 \
  -e BEARER_TOKEN=replace_with_test_token \
  scripts/k6/seat-contention.js
~~~

Để hoàn thành 10 lượt/1.000 request, chuẩn bị **10 ghế benchmark còn trống** trên suất diễn test; chạy script 10 lần, mỗi lần đổi SEAT_ID. Không dùng một ghế đã được lượt trước giữ. Nếu chưa thêm authentication, bỏ BEARER_TOKEN.

Trong bài này, 409 là kết quả nghiệp vụ mong đợi cho 99 request thua cuộc, không phải lỗi server. Script đánh dấu 201/409 là status mong đợi cho metric HTTP, nhưng vẫn kiểm tra 409 có đúng code SEAT_UNAVAILABLE hay không; mọi status khác bị tính là bất thường. k6 hỗ trợ đặt status mong đợi, threshold và check; các threshold làm lượt test fail nếu số thành công/xung đột sai hoặc có response lạ. Xem [k6 expected statuses](https://grafana.com/docs/k6/latest/javascript-api/k6-http/expected-statuses/), [k6 thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/) và [k6 scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/).

**Diễn giải kết quả:** 1 thành công + 99 xung đột chứng minh invariant “không bán trùng” trong kịch bản này; nó không có nghĩa 99% request bị lỗi. Nếu số thành công là 0 hoặc lớn hơn 1, test fail. Nếu có 5xx, kiểm tra log ứng dụng, lock timeout, deadlock và connection pool trước khi chạy lại.

### 15.5 Benchmark API đọc và latency

Dùng một suất diễn có **100 ghế** và dữ liệu event/showtime giống nhau giữa các lần chạy. Đo riêng GET /api/showtimes/{id}/seats; đừng trộn với bài contention vì 99 response 409 làm sai ý nghĩa latency của API đọc.

Kịch bản ban đầu:

| Mức tải | Tốc độ đến mục tiêu | Thời lượng đo | Lặp |
|---|---:|---:|---:|
| Nhẹ | 10 request/giây | 3 phút | 3 lần |
| Vừa | 25 request/giây | 3 phút | 3 lần |
| Cao hơn | 50 request/giây | 3 phút | 3 lần |

Tạo file scripts/k6/seat-list.js:

~~~javascript
import http from 'k6/http';
import { check } from 'k6';

const baseUrl = __ENV.BASE_URL || 'http://localhost:8080';
const showtimeId = __ENV.SHOWTIME_ID;
const bearerToken = __ENV.BEARER_TOKEN;
const targetRate = Number(__ENV.TARGET_RPS || 25);
const duration = __ENV.DURATION || '3m';

export const options = {
  scenarios: {
    seat_list: {
      executor: 'constant-arrival-rate',
      rate: targetRate,
      timeUnit: '1s',
      duration,
      preAllocatedVUs: 50,
      maxVUs: 100,
    },
  },
  thresholds: {
    'http_req_duration{endpoint:seat-list}': ['p(95)<300'],
    'http_req_failed{endpoint:seat-list}': ['rate<0.01'],
    dropped_iterations: ['count==0'],
    checks: ['rate==1'],
  },
};

export default function () {
  const headers = {};
  if (bearerToken) {
    headers.Authorization = 'Bearer ' + bearerToken;
  }

  const response = http.get(
    baseUrl + '/api/showtimes/' + showtimeId + '/seats',
    { headers, tags: { endpoint: 'seat-list' } },
  );

  check(response, {
    'seat list returns HTTP 200': (res) => res.status === 200,
  });
}
~~~

Trước mỗi lượt đo, warm-up 1 phút ở cùng TARGET_RPS. Không đưa lượt warm-up vào bảng kết quả:

~~~bash
k6 run \
  -e BASE_URL=http://localhost:8080 \
  -e SHOWTIME_ID=9001 \
  -e TARGET_RPS=25 \
  -e DURATION=1m \
  -e BEARER_TOKEN=replace_with_test_token \
  scripts/k6/seat-list.js
~~~

Sau đó chạy lượt đo 3 phút:

~~~bash
k6 run \
  -e BASE_URL=http://localhost:8080 \
  -e SHOWTIME_ID=9001 \
  -e TARGET_RPS=25 \
  -e DURATION=3m \
  -e BEARER_TOKEN=replace_with_test_token \
  scripts/k6/seat-list.js
~~~

Lặp cả cặp warm-up/đo 3 lần. Đổi TARGET_RPS lần lượt thành 10, 25 và 50. Dùng showtime ổn định có 100 ghế; API đọc không làm thay đổi dữ liệu. Constant arrival rate nhắm tới số iteration bắt đầu mỗi giây; nếu k6 không đủ VU để theo kịp, dropped_iterations sẽ báo tải không đạt. Chạy riêng mỗi mức để biết p95 nào ứng với RPS nào.

Mục tiêu dev khởi điểm ở mức vừa: **p95 ≤ 300 ms**, **5xx = 0**, không có iteration bị rơi. Nếu máy dev chưa đạt, ghi p95/RPS thực tế và phần cứng; đây là mục tiêu để điều tra, không phải số được phép đưa vào CV khi chưa đo.

Định nghĩa:

- **RPS:** số response hoàn tất chia thời gian đo; ghi cả target arrival rate và RPS thực tế.
- **p95:** 95% request có latency bằng hoặc thấp hơn giá trị này.
- **p99:** 99% request có latency bằng hoặc thấp hơn giá trị này; giữ làm số tham khảo khi tải cao.
- **Tỷ lệ lỗi kỹ thuật:** 5xx / tổng request; báo riêng các 4xx nghiệp vụ dự kiến.
- **Dropped iterations:** request k6 dự kiến gửi nhưng không được tạo đúng lịch; nếu lớn hơn 0 thì tải thực tế không bằng mục tiêu.

### 15.6 So sánh connection pool 5 và 10

Chỉ làm sau khi API đúng. Trong application.yml, cho phép thay pool size qua env:

~~~yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      minimum-idle: 2
      connection-timeout: 3000
~~~

Giữ nguyên máy, dữ liệu, commit, Docker resources và workload. Dùng benchmark API đọc mức **25 request/giây × 3 phút**, warm-up 1 phút, lặp **3 lần** mỗi pool. Chạy pool 5 và pool 10; nếu database bị nghẽn connection, CPU/RAM cao hoặc lock wait tăng thì không thử lớn hơn chỉ để tìm con số cao.

Đo và ghi:

- p95/p99 latency.
- RPS thực tế và dropped iterations.
- 5xx.
- Số connection PostgreSQL lớn nhất trong khi chạy.
- CPU/RAM ứng dụng và PostgreSQL, nếu có công cụ đo.

Truy vấn số connection theo trạng thái trong PostgreSQL:

~~~sql
SELECT state, count(*) AS connections
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state
ORDER BY state;
~~~

Chọn cấu hình theo p95 và khả năng DB, không chọn theo RPS một mình. Pool của tất cả app instance cộng lại không được vượt ngân sách connection PostgreSQL.

### 15.7 Thử nghiệm index có đối chứng

Đo một truy vấn thực của API danh sách suất diễn:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, event_id, start_time, end_time
FROM showtimes
WHERE event_id = 42
ORDER BY start_time;
~~~

Tạo dữ liệu dev/benchmark gồm **500 event × 40 showtime = 20.000 showtime**. Chạy ANALYZE showtimes;, đo truy vấn trước index và lưu query plan, thời gian thực, buffers.

Ghi rõ baseline: migration V1 đã tạo GiST index phục vụ exclusion constraint của showtimes. Thử nghiệm này so sánh việc bổ sung B-tree (event_id, start_time), không phải so sánh “không có bất kỳ index nào” với “có index”. Sau khi lưu baseline, thêm migration V2__index_showtimes_by_event_time.sql:

~~~sql
CREATE INDEX showtimes_event_start_idx
    ON showtimes (event_id, start_time);
~~~

Khởi động lại app để Flyway chạy V2, chạy lại ANALYZE showtimes; và đúng truy vấn trên. So sánh plan có dùng index không, execution time và buffers. Nếu planner vẫn chọn sequential scan vì dữ liệu nhỏ/chi phí thấp, ghi nhận đúng kết quả; không tuyên bố index nhanh hơn chỉ vì index đã được tạo.

Chỉ thử xóa/thêm index trong database benchmark có thể dựng lại; không bỏ unique index hoặc constraint bảo vệ dữ liệu để làm benchmark. Thay đổi schema chính thức phải đi qua migration.

### 15.8 Cách chạy phép đo công bằng

Trước mỗi nhóm thử nghiệm, lưu:

| Thông tin | Giá trị cần ghi |
|---|---|
| Ngày giờ, git commit | |
| Java, Spring Boot, PostgreSQL, k6 | |
| CPU, RAM, hệ điều hành | |
| Ứng dụng chạy trên host hay container | |
| Dataset và số hàng | |
| Pool size | |
| Kịch bản, VU/RPS mục tiêu, thời gian | |
| Số lần lặp | |
| p50 / p95 / p99 | |
| RPS thực tế / dropped iterations | |
| 201 / 409 / 5xx | |
| Duplicate active seat count | |
| Ghi chú về CPU, connection, lock wait | |

Quy trình:

1. Dùng cùng commit và cùng fixture cho các lượt cần so sánh.
2. Chạy warm-up; không gộp warm-up vào thời gian đo.
3. Chạy mỗi cấu hình **3 lần** và giữ output từng lần; báo median của 3 lần, không chỉ chọn lần đẹp nhất.
4. Khi đổi đúng một yếu tố (pool/index), giữ nguyên các yếu tố còn lại.
5. Ghi rõ app và DB chạy trên cùng laptop hay máy riêng. Load generator chạy cùng laptop có thể tranh CPU với app/DB.
6. Không xóa một kết quả chậm/lỗi khỏi báo cáo; ghi nguyên nhân và lượt chạy lại riêng.

### 15.9 Bảng kết quả thực tế

Điền bảng này sau khi chạy. Các ô trống là số liệu cần thu thập, không được thay bằng mục tiêu ở trên.

Kết quả integration test xác định race:

| Cách xử lý | Ghế | Lần thử | Hold được tạo | Conflict | Hold vượt mức | Kết quả thực đo |
|---|---:|---:|---:|---:|---:|---|
| Baseline test-only | 10 | 500 transaction | | | | |
| Implementation thật | 10 | 500 transaction | | | | |

Kết quả k6 HTTP và benchmark:

| Thử nghiệm | Run | Cấu hình | Request | 201 | 409 | 5xx | Duplicate | p95 | RPS thực tế | Dropped |
|---|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Cùng 1 ghế | 1 | 100 VU | 100 | | | | | | | |
| Cùng 1 ghế | 2 | 100 VU | 100 | | | | | | | |
| Cùng 1 ghế | 3 | 100 VU | 100 | | | | | | | |
| Cùng 1 ghế | … | 10 ghế độc lập | 1.000 tổng | | | | | | | |
| Đọc ghế | 1 | 25 req/s, pool 5 | | | n/a | | n/a | | | |
| Đọc ghế | 1 | 25 req/s, pool 10 | | | n/a | | n/a | | | |
| Đọc suất diễn | Trước | Chưa có B-tree index | | n/a | n/a | n/a | n/a | | n/a | n/a |
| Đọc suất diễn | Sau | Có V2 composite index | | n/a | n/a | n/a | n/a | | n/a | n/a |

Với hàng 1.000 request, ghi tổng của đủ 10 lượt. Kết quả đạt mục tiêu đúng kỳ vọng là 201 = 10, 409 = 990, 5xx = 0, Duplicate = 0. Chỉ điền các giá trị đó sau khi output test và truy vấn DB xác nhận.

---

## 16. Docker hóa ứng dụng sau khi chạy tốt trên máy

Bước này có thể làm sau MVP. Khi app cũng chạy trong Compose:

- Thêm service app dùng image build từ Dockerfile.
- App kết nối PostgreSQL qua hostname service postgres, không dùng localhost.
- URL trong container có dạng jdbc:postgresql://postgres:5432/datve_db.
- Thêm dependency condition đợi postgres healthy.
- Chỉ expose cổng app cần thiết; DB không nên mở ra Internet trên production.
- Dùng biến môi trường/secrets cho credential. Không copy .env hoặc credential production vào image.
- Thêm .dockerignore để bỏ target/, .git/, .env, file IDE.
- Dùng non-root user trong container khi đóng gói production.

Trên máy dev chạy app ngoài container thì vẫn dùng localhost:5432. Hai địa chỉ này khác nhau vì localhost bên trong container là chính container đó.

---

## 17. Lỗi thường gặp và cách tìm nguyên nhân

| Triệu chứng | Nguyên nhân hay gặp | Cách xử lý |
|---|---|---|
| Connection refused ở port 5432 | PostgreSQL chưa chạy, sai port hoặc container chưa healthy | docker compose ps, docker compose logs postgres; chạy docker compose up -d postgres |
| password authentication failed | Username/password của Spring không khớp với container đã khởi tạo | Kiểm tra .env, biến DB_*; volume cũ giữ credential cũ, đổi env không tự đổi password của DB hiện có |
| database does not exist | DB_URL trỏ sai tên database | Kiểm tra URL và POSTGRES_DB |
| Spring không đọc .env | Spring Boot không tự nạp file .env | Dùng env var trong shell/IDE hoặc đặt default dev trong application.yml; Compose tự đọc .env cho cấu hình Compose |
| No suitable driver | Thiếu PostgreSQL Driver hoặc dependency chưa tải | Xem pom.xml, reload Maven trong IDE |
| Flyway không nhận PostgreSQL | Thiếu Flyway database module hoặc dependency không tương thích | Chọn dependency Flyway PostgreSQL do Initializr/Spring Boot quản lý |
| permission denied to create extension | User database không có quyền tạo extension | Dùng database owner trên local; môi trường managed cần DBA bật btree_gist |
| Flyway báo checksum mismatch | File migration đã sửa sau khi chạy | Khôi phục file cũ và tạo migration mới; chỉ reset local DB nếu dữ liệu bỏ được |
| Hibernate Schema-validation lỗi | Entity chưa khớp schema Flyway | So sánh kiểu dữ liệu, tên cột, nullable và migration; không chuyển ddl-auto sang update để che lỗi |
| Port 8080 đã được sử dụng | App khác đang nghe port đó | Dừng app kia hoặc chạy với SERVER_PORT=8081 |
| Dùng localhost trong app container không kết nối DB | localhost là chính app container | Dùng hostname Compose postgres |
| Booking giữ ghế hết hạn nhưng ghế vẫn unavailable | Chưa chạy cleanup hoặc API chưa giải phóng hold hết hạn trước khi giữ | Kiểm tra worker và transaction của use case giữ ghế |

---

## 18. Lộ trình thực thi theo chặng

| Chặng | Việc hoàn tất | Kết quả có thể kiểm tra |
|---|---|---|
| 0 | Cài JDK, Docker, Git; tạo Spring project | mvnw spring-boot:run biên dịch và bắt đầu chạy |
| 1 | Compose PostgreSQL, cấu hình datasource | Spring kết nối được database |
| 2 | Flyway V1 và kiểm tra schema | flyway_schema_history có V1 thành công |
| 3 | Event/showtime/seat đọc dữ liệu | GET API trả JSON từ PostgreSQL |
| 4 | Tạo hold transaction và unique index | Request cạnh tranh cùng ghế chỉ có một request thành công |
| 5 | Cancel và expiry worker | Ghế được giải phóng và có thể đặt lại |
| 6 | Mock payment + idempotency | Luồng success/fail cập nhật booking, payment, seat nhất quán |
| 7 | Security, role, quyền sở hữu booking | Endpoint admin và user được bảo vệ |
| 8 | Integration/concurrency test trên PostgreSQL thật | Test 100 request cùng ghế; lưu log và truy vấn invariant |
| 9 | Đo latency, thử pool/index, Docker hóa app | Bảng trước/sau có môi trường, workload và số đo |
| 10 | README, portfolio và public demo nếu phù hợp | Recruiter clone/chạy được; không lộ secret |

### Definition of Done cho MVP

- Người mới clone repo có thể làm theo README để chạy app và PostgreSQL.
- Không cần tạo bảng thủ công; Flyway dựng schema.
- API đọc event/showtime/seat từ PostgreSQL.
- Hai request cạnh tranh cùng ghế không thể cùng thành công.
- Booking hết hạn/hủy/thanh toán thất bại giải phóng ghế đúng cách.
- Thanh toán thành công giữ ghế ở trạng thái BOOKED.
- Lỗi API có HTTP status và JSON nhất quán.
- Credential local/prod không bị commit.
- Kịch bản 100 request/ghế lặp 10 lượt cho kết quả mục tiêu: 10 thành công, 990 conflict, 0 lỗi 5xx, 0 ghế active trùng; các số đã được đo và lưu, không chỉ để ở dạng mục tiêu.
- Có bảng benchmark latency/pool/index với 3 lượt mỗi cấu hình, ghi đủ phần cứng, dataset, commit và RPS thực tế.
- README cho phép người khác chạy dự án; CV chỉ dùng số đã đối chiếu với output test và DB.

---

## 19. Tài liệu chính thức

- [Spring Initializr](https://start.spring.io/)
- [Spring Boot: System Requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Boot: SQL Databases](https://docs.spring.io/spring-boot/reference/data/sql.html)
- [Spring Boot: Actuator](https://docs.spring.io/spring-boot/reference/actuator/index.html)
- [PostgreSQL: Range Types và exclusion constraint](https://www.postgresql.org/docs/current/rangetypes.html)
- [PostgreSQL: btree_gist](https://www.postgresql.org/docs/current/btree-gist.html)
- [PostgreSQL: Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [Docker Compose: PostgreSQL quickstart](https://docs.docker.com/guides/postgresql/)
- [Docker Engine on Ubuntu and Ubuntu derivatives](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Engine on Debian and Debian derivatives](https://docs.docker.com/engine/install/debian/)
- [Eclipse Adoptium: Linux installation](https://adoptium.net/installation/linux/)
- [k6: Thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
- [k6: Scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/)
- [k6: Expected HTTP statuses](https://grafana.com/docs/k6/latest/javascript-api/k6-http/expected-statuses/)
- [Testcontainers for Java: PostgreSQL module](https://java.testcontainers.org/modules/databases/postgres/)

---

## 20. Chuyển kết quả thành CV và portfolio

CV cần nói được bốn ý theo thứ tự: **bài toán**, **cách xử lý**, **số đo**, **ý nghĩa với hệ thống**. Với dự án này:

- Bài toán: nhiều người có thể chọn cùng ghế trong một thời điểm.
- Cách xử lý: transaction, khóa hàng PostgreSQL và partial unique index; hold hết hạn được giải phóng.
- Chỉ số chính: số booking/hold active trùng ghế.
- Ý nghĩa: không bán trùng ghế, giảm hoàn tiền/đổi chỗ và khiếu nại.

### Mẫu bullet CV — chỉ điền sau khi đo thật

**Bản tập trung vào tính đúng đắn:**

> Xây dựng backend đặt vé bằng Spring Boot/PostgreSQL; xử lý tranh chấp ghế bằng transaction, row lock và partial unique index. Trong bài kiểm thử **[N] request đồng thời/ghế × [R] lượt**, ghi nhận **[W] booking thắng, [C] xung đột hợp lệ, [E] lỗi 5xx và [D] ghế bị đặt trùng**; mục tiêu là D = 0.

Với đúng kế hoạch benchmark ở mục 15, nếu cả 10 lượt đều đạt, có thể điền N = 100, R = 10, W = 10, C = 990, E = 0, D = 0. Chỉ dùng các số đó khi log k6 và DB query đều xác nhận.

**Bullet nêu rõ trước/sau cho bài toán concurrency:**

> Trong test race có barrier với 50 transaction/ghế × 10 ghế, giảm số hold vượt mức từ **[baseline đo được]** xuống **[sau khi thêm row lock + unique index]**; implementation thật trả **[số]** conflict hợp lệ và duy trì **0** ghế đặt trùng.

Theo fixture mục 15, kết quả kỳ vọng là **490 → 0 hold vượt mức**. Khi đưa lên CV, ghi đây là *controlled barrier test*; không diễn giải con số đó thành tỷ lệ lỗi của người dùng production.

**Bản tập trung vào tối ưu hiệu năng, chỉ dùng khi có so sánh trước/sau:**

> Tối ưu API **[endpoint]** từ p95 **[A] ms** xuống **[B] ms** (**[P]%**) ở **[RPS] request/giây**, bằng **[index/pool/query cụ thể]**; đo trên **[CPU/RAM, PostgreSQL version, dataset]**, mỗi cấu hình 3 lượt.

Công thức giảm latency: **(A − B) / A × 100%**. Công thức tăng throughput: **(sau − trước) / trước × 100%**. Không dùng từ “tối ưu” nếu chưa lưu số trước/sau cùng workload.

### Cách giải thích trong phỏng vấn

1. Nêu hậu quả cụ thể: bán trùng ghế khiến hệ thống phải đổi chỗ/hoàn tiền và mất niềm tin.
2. Nêu cách dựng lỗi: gửi N request cùng chọn một ghế, không chỉ nói chung chung “có concurrency”.
3. Nêu invariant và kết quả thực đo: một booking thành công, các request còn lại nhận conflict, không có ghế trùng.
4. Nêu vì sao cần cả transaction/row lock lẫn constraint DB: DB bảo vệ invariant kể cả khi code path có lỗi.
5. Nêu giới hạn: con số chỉ đại diện cho máy, dataset và workload đã ghi; không tự suy ra được quy mô production.

### Checklist trước khi public project

- Repo công khai có README tiếng Việt hoặc tiếng Anh với lệnh khởi động từ đầu.
- Có ERD, mô tả state machine booking/seat/payment và OpenAPI/collection request.
- Có hướng dẫn chạy PostgreSQL/Flyway và một kịch bản benchmark có thể tái lập.
- Đưa output thật của k6, bảng kết quả và phần cứng vào thư mục docs/performance; không đưa token/mật khẩu.
- Nếu deploy demo public: chỉ dùng payment mock, dữ liệu giả, tài khoản demo quyền hạn thấp và giới hạn tài nguyên; không mở database hay Actuator nhạy cảm ra Internet.
- Ghi link repo/demo và 2–3 dòng kết quả nổi bật vào CV; chuẩn bị mở được project và giải thích số đo khi phỏng vấn.

---

## Việc tiếp theo sau khi đọc tài liệu

Làm đúng thứ tự các mục **2 → 7** trước. Khi /actuator/health báo UP và flyway_schema_history có migration V1 thành công, bắt đầu mục **8 → 10** để có API đọc event và giữ ghế. Sau khi luồng nghiệp vụ đúng, làm test ở mục 15, ghi số đo thật, rồi hoàn thiện README/portfolio ở mục 20. Chưa thêm Redis, RabbitMQ hoặc gateway thanh toán thật trước khi luồng DB cơ bản chạy đúng.
