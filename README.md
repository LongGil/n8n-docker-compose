# ⚙️ Kiến Trúc Máy Chủ Tự Động Hóa GDD - GilongWorld

Tài liệu này hướng dẫn triển khai hệ thống máy chủ tự động hóa cục bộ (Local-First) quản lý và khởi tạo nội dung Game Design Document (GDD) cho dự án AAA Indie **GilongWorld**.

Hệ thống sử dụng **n8n** chạy trên môi trường **Ubuntu**, giao tiếp bảo mật qua **Cloudflare Tunnel** và tự động xuất các bản thiết kế hệ thống (Monster Taming, Base Building, Alchemy/Potion Crafting) ra định dạng **Markdown** trực tiếp trên ổ cứng để đồng bộ vào Game Engine.

---

## 1. Cài Đặt Môi Trường Cơ Bản (Ubuntu)

Cập nhật danh sách gói hệ thống và cài đặt nền tảng Docker cùng trình soạn thảo nano:

```bash
sudo apt update
sudo apt install docker.io docker-compose nano -y
```

---

## 2. Khởi Tạo Thư Mục Và Phân Quyền (Bắt buộc)

Tạo các thư mục vật lý để lưu trữ database lõi của n8n và các file Markdown thiết kế game.
Cần cấp quyền sở hữu cho user ảo của Docker (UID 1000) để tránh lỗi Permission Denied.

```bash
# Tạo thư mục chứa cấu hình n8n và thư mục chứa GDD của GilongWorld
mkdir -p ~/n8n-data
mkdir -p ~/local-files

# Ép phân quyền cho Docker
sudo chown -R 1000:1000 ~/n8n-data
sudo chown -R 1000:1000 ~/local-files
```

---

## 3. Cấu Hình Hệ Thống (Docker Compose)

Mở trình soạn thảo để tạo tệp cấu hình hạ tầng:

```bash
nano docker-compose.yml
```

Dán toàn bộ nội dung cấu hình sau vào tệp:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_gilongworld
    restart: always
    ports:
      - "5678:5678"
    environment:
      # --- 1. MÔI TRƯỜNG & BẢO MẬT ---
      - NODE_ENV=production
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      # --- 2. KẾT NỐI MẠNG (WEBHOOK QUA CLOUDFLARE TUNNEL) ---
      - N8N_HOST=n8n.longgilstudio.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.longgilstudio.com/
      # --- 3. THỜI GIAN HỆ THỐNG ---
      - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
      - TZ=Asia/Ho_Chi_Minh
      # --- 4. TỐI ƯU Ổ CỨNG (TỰ DỌN LOG) ---
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168 # Xóa dữ liệu cũ hơn 7 ngày
    volumes:
      # --- 5. BIND MOUNT (QUẢN LÝ DỮ LIỆU LOCAL) ---
      - ~/n8n-data:/home/node/.n8n
      - ~/local-files:/files
```

*(Lưu tệp bằng cách nhấn `Ctrl + O` → `Enter`, sau đó nhấn `Ctrl + X` để thoát).*

---

## 4. Bộ Lệnh Quản Trị Vận Hành

Sử dụng các lệnh dưới đây (tại thư mục chứa file `docker-compose.yml`) để điều khiển máy chủ tự động hóa:

### ▶️ Khởi động hệ thống (Chạy ngầm)

```bash
sudo docker-compose up -d
```

### ⏸️ Tắt hệ thống hoàn toàn (Dữ liệu vẫn được bảo toàn)

```bash
sudo docker-compose down
```

### 🔄 Cập nhật và áp dụng cấu hình mới

```bash
sudo docker-compose up -d --force-recreate
```

### 🚀 Cập nhật n8n lên phiên bản mới nhất

```bash
sudo docker-compose pull
sudo docker-compose up -d
```

### 🩺 Kiểm tra Log hệ thống

```bash
sudo docker logs -f n8n_gilongworld
```

---

## 5. Dọn Rác Máy Chủ Định Kỳ (Cronjob)

Mở bảng cấu hình lịch trình của Ubuntu:

```bash
sudo crontab -e
```

Thêm dòng lệnh sau vào cuối file:

```plaintext
0 3 * * 0 /usr/bin/docker system prune -af --volumes
```

Tác vụ này sẽ tự động xóa các image dư thừa của Docker vào **03:00 sáng Chủ Nhật hàng tuần**.
