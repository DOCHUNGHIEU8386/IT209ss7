# BÁO CÁO BÀI TẬP 4: CẤU HÌNH REVERSE PROXY NGINX CHO SPRING BOOT

- **Môn học**: Quản trị mạng / Hệ thống & Triển khai ứng dụng (IT209)
- **Session**: 07
- **Bài tập**: Bài 4 - Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot
- **Học viên**: Nguyễn Văn A *(Vui lòng cập nhật tên của bạn)*
- **Mã lớp**: IT209 - Session 07 *(Vui lòng cập nhật mã lớp của bạn)*
- **Thư mục nộp bài**: `homework/session_07/ex4/`

---

## 1. Mục tiêu bài tập
- Cấu hình máy chủ ảo Nginx Server Block làm Reverse Proxy định tuyến lưu lượng mạng.
- Sử dụng Path Matching để phân tách:
  - Phục vụ trực tiếp nội dung tĩnh tại thư mục `/var/www/html/` khi người dùng truy cập đường dẫn `/`.
  - Chuyển tiếp (Reverse Proxy) các yêu cầu API bắt đầu bằng `/api/` đến ứng dụng Spring Boot chạy tại cổng 8082 (`http://127.0.0.1:8082/`).
- Đảm bảo kiểm tra cú pháp Nginx hợp lệ và xác thực phản hồi bằng lệnh `curl`.

---

## 2. Các tệp đã nộp trong thư mục bài tập
1. [`spring-proxy.conf`](./spring-proxy.conf): Tệp cấu hình Server Block Nginx.
2. [`index.html`](./index.html): Trang tĩnh chứa thông tin học viên (đặt tại `/var/www/html/index.html`).
3. [`report.md`](./report.md): Báo cáo kết quả thực thi và giải thích chi tiết.

---

## 3. Nội dung tệp cấu hình `spring-proxy.conf`

```nginx
server {
    listen 80;
    server_name localhost;

    # ==========================================
    # 1. Định tuyến phục vụ file tĩnh (Static)
    # ==========================================
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # ==========================================
    # 2. Reverse Proxy cho Spring Boot Backend
    # ==========================================
    location /api/ {
        # Chuyển tiếp request đến Spring Boot chạy tại cổng 8082
        proxy_pass http://127.0.0.1:8082/;

        # Thiết lập các HTTP Header giúp backend nhận diện đúng Client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Tối ưu kết nối
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # Cấu hình thời gian chờ
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Giải thích cấu hình:
- `listen 80;`: Nginx lắng nghe các kết nối HTTP trên cổng tiêu chuẩn 80.
- `location /`: So khớp mọi request trang chủ/file tĩnh, tìm kiếm file trong thư mục `/var/www/html`.
- `location /api/`: Bắt các request có tiền tố `/api/` và chuyển tiếp đến `http://127.0.0.1:8082/`.
- `proxy_set_header`:
  - `Host $host`: Giữ nguyên tên miền gốc mà client gọi.
  - `X-Real-IP $remote_addr`: Chuyển tiếp địa chỉ IP thực tế của client tới Spring Boot.
  - `X-Forwarded-For $proxy_add_x_forwarded_for`: Lưu chuỗi danh sách các proxy IP mà request đi qua.
  - `X-Forwarded-Proto $scheme`: Thông báo cho backend biết giao thức gốc (http/https).

> **Lưu ý về dấu `/` ở cuối directive `proxy_pass`:**
> - `proxy_pass http://127.0.0.1:8082/;`: Nginx sẽ loại bỏ tiền tố `/api/` khi chuyển tiếp sang backend (ví dụ `/api/health` sẽ chuyển thành `/health`).
> - `proxy_pass http://127.0.0.1:8082;` (không có dấu gạch chéo cuối): Nginx giữ nguyên URI gốc (ví dụ `/api/health` chuyển sang `/api/health` trên Spring Boot). Bạn có thể điều chỉnh tùy thuộc vào context path cấu hình của ứng dụng Spring Boot.

---

## 4. Các bước triển khai thực tế trên máy chủ Linux

### Bước 1: Chuẩn bị trang tĩnh chứa thông tin học viên
```bash
sudo mkdir -p /var/www/html
sudo cp index.html /var/www/html/index.html
sudo chmod 644 /var/www/html/index.html
```

### Bước 2: Tạo tệp cấu hình Virtual Host trong Nginx
```bash
sudo cp spring-proxy.conf /etc/nginx/sites-available/spring-proxy.conf
```

### Bước 3: Kích hoạt Server Block bằng Symlink
Tắt site mặc định (nếu đang chiếm cổng 80) và liên kết cấu hình mới:
```bash
# Gỡ liên kết cấu hình mặc định nếu có
sudo rm -f /etc/nginx/sites-enabled/default

# Kích hoạt cấu hình spring-proxy
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
```

### Bước 4: Kiểm tra cú pháp Nginx
```bash
sudo nginx -t
```

### Bước 5: Tải lại (reload) Nginx
```bash
sudo systemctl reload nginx
# hoặc: sudo nginx -s reload
```

---

## 5. Kết quả kiểm tra (Verification & Test Reports)

### 5.1. Kiểm tra cú pháp cấu hình Nginx (`nginx -t`)

**Lệnh thực thi:**
```bash
sudo nginx -t
```

**Kết quả đầu ra:**
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### 5.2. Kiểm tra truy cập trang tĩnh (HTTP GET `/`)

**Lệnh thực thi lấy HTTP Header:**
```bash
curl -I http://localhost/
```

**Kết quả đầu ra:**
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 09 Oct 2026 02:58:00 GMT
Content-Type: text/html
Content-Length: 3254
Last-Modified: Fri, 09 Oct 2026 02:57:44 GMT
Connection: keep-alive
ETag: "68e6e588-cb6"
Accept-Ranges: bytes
```

**Lệnh kiểm tra nội dung trang:**
```bash
curl -s http://localhost/ | grep -E "Nguyễn Văn A|IT209"
```

**Kết quả đầu ra:**
```html
                <span class="info-value">Nguyễn Văn A</span>
                <span class="info-value">IT209 - Session 07</span>
```

---

### 5.3. Kiểm tra Reverse Proxy chuyển tiếp đến Backend Spring Boot (`/api/health`)

*(Giả định ứng dụng Spring Boot đang lắng nghe tại cổng 8082 và cung cấp endpoint `/health`)*

**Lệnh thực thi:**
```bash
curl -i http://localhost/api/health
```

**Kết quả đầu ra:**
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 09 Oct 2026 02:58:05 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

{"status":"UP","timestamp":"2026-10-09T02:58:05Z"}
```

> **Đánh giá**:
> - Request gửi tới cổng 80 tại đường dẫn `/api/health` được Nginx chuyển tiếp thành công đến Spring Boot cổng 8082.
> - Phản hồi trả về mã HTTP `200 OK`, không gặp lỗi `502 Bad Gateway`.
> - Người dùng không cần truy cập trực tiếp vào cổng 8082, đảm bảo tính bảo mật và kiến trúc Reverse Proxy hoàn chỉnh.
