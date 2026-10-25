# ĐỀ TÀI 29 — RECIPEHUB: WEBSITE CHIA SẺ CÔNG THỨC NẤU ĂN

Triển khai, đóng gói, điều phối, giám sát và bảo mật hệ thống **Website chia sẻ công thức nấu ăn (RecipeHub)** bằng **Docker Compose** — Bài tập môn **Triển khai và Quản trị Hệ thống Phần mềm**.

## 👨‍💻 Thông tin sinh viên thực hiện

| Thông tin | Nội dung |
| --- | --- |
| **Trường** | Trường Đại học Công nghệ Thông tin và Truyền thông (ICTU) — Khoa Công nghệ Thông tin |
| **Học phần** | Triển khai và Quản trị Hệ thống Phần mềm |
| **Giảng viên hướng dẫn** | TS. Nguyễn Anh Chuyên |
| **Họ và tên** | Hoàng Công Sự |
| **MSSV** | DTC245160042 |
| **Đề tài** | Đề 29 — Website Công thức Nấu ăn / Recipe (Món ăn, nguyên liệu, danh mục) |
| **Repository** | https://github.com/hight-87ii/dtc245160042-HoangCongSu |

---

## 🚀 1. Tổng quan hệ thống & Công nghệ sử dụng (Tech Stack)

Hệ thống được triển khai hoàn toàn bằng **Docker Compose** theo kiến trúc multi-container, gồm **11 dịch vụ** kết nối qua **3 phân vùng mạng độc lập** (`frontend_net`, `backend_net`, `monitoring_net`).

| Thành phần | Công nghệ / Image | Container | Vai trò trong hệ thống |
| --- | --- | --- | --- |
| **Reverse Proxy** | `nginx:latest` | `recipe-nginx` | Điều hướng truy cập cổng `80` vào WordPress, thiết lập HTTP Security Headers. |
| **Web Application** | `wordpress:latest` | `recipe-wordpress` | CMS vận hành website chia sẻ công thức nấu ăn **RecipeHub**. |
| **Database** | `mysql:8.0` | `recipe-mysql` | CSDL quan hệ lưu bài viết món ăn, nguyên liệu, danh mục và người dùng. |
| **DB Management** | `phpmyadmin/phpmyadmin:latest` | `recipe-phpmyadmin` | Giao diện web quản trị trực tiếp MySQL. |
| **Metrics Collector** | `prom/prometheus:latest` | `recipe-prometheus` | Thu thập và lưu trữ time-series metrics từ các exporter. |
| **Visualization** | `grafana/grafana:latest` | `recipe-grafana` | Dashboard giám sát hạ tầng, container, DB và truy vấn log tập trung. |
| **Host Exporter** | `prom/node-exporter:latest` | `recipe-node-exporter` | Thu thập thông số tài nguyên máy chủ Linux (CPU, RAM, disk, network). |
| **Container Exporter** | `gcr.io/cadvisor/cadvisor:latest` | `recipe-cadvisor` | Thu thập metrics tài nguyên của từng Docker container. |
| **MySQL Exporter** | `prom/mysqld-exporter:latest` | `recipe-mysql-exporter` | Thu thập metrics của MySQL Server. |
| **Log Storage** | `grafana/loki:latest` | `recipe-loki` | Lưu trữ và lập chỉ mục log tập trung. |
| **Log Collector** | `grafana/promtail:latest` | `recipe-promtail` | Tự động thu thập log toàn bộ container qua Docker Socket. |

### Kiến trúc mạng

```
                    ┌──────────────────────────── INTERNET / CLIENT ────────────────────────────┐
                    │                           :80        :8081     :3000   :9090   :3100    │
                    └─────────────────────────────┬────────────┬─────────┬───────┬───────┬────┘
                                                  │            │         │       │       │
   ┌─────────────────────────── frontend_net ─────┼────────────┼─────────┼───────┼───────┼────┐
   │                                              │            │         │       │       │    │
   │   recipe-nginx  ──────────────────────►  recipe-wordpress     recipe-phpmyadmin          │
   │                                              │  │                                           
   └──────────────────────────────────────────────┼──┼────────────────────────────────────────┘
                                                  │  │
   ┌─────────────────────────── backend_net ──────┼──┼────────────────────────────────────────┐
   │                                              ▼  ▼                                          │
   │   recipe-mysql  ◄──── recipe-mysql-exporter        (MySQL KHÔNG publish port ra host)       │
   └──────────────────────────────────────────────────────────────────────────────────────────┘
                                                  │
   ┌─────────────────────────── monitoring_net ───┼───────────────────────────────────────────┐
   │   recipe-prometheus ──┬── recipe-node-exporter                                             │
   │                       ├── recipe-cadvisor                                                 │
   │                       └── recipe-mysql-exporter                                           │
   │   recipe-grafana ──► prometheus + loki                                                     │
   │   recipe-promtail ──► recipe-loki                                                          │
   └──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📂 2. Cấu trúc thư mục dự án

```
dtc245160042-HoangCongSu/
├── docker-compose.yml          # Điều phối 11 dịch vụ, 3 mạng bridge và 5 volumes
├── .env                        # Biến môi trường nhạy cảm (KHÔNG đẩy lên Git)
├── .gitignore                  # Bỏ qua .env và *.log
├── nginx/
│   └── default.conf            # Reverse Proxy + HTTP Security Headers
├── prometheus/
│   └── prometheus.yml          # Cấu hình scrape targets (node / cadvisor / mysql exporter)
├── loki/
│   └── loki-config.yaml        # Cấu hình lưu trữ và lập chỉ mục log
├── promtail/
│   └── promtail-config.yml     # Cấu hình agent thu thập log qua Docker socket
└── README.md
```

---

## 🛠️ 3. Hướng dẫn cài đặt và triển khai

### 3.1. Yêu cầu môi trường

- Hệ điều hành: **Ubuntu Server 20.04 / 22.04 LTS** hoặc Linux tương đương.
- **Docker Engine** và **Docker Compose v2.x** trở lên.
- **Git**.
- Cổng `80`, `3000`, `3100`, `8081`, `9090` chưa bị chiếm dụng.

### 3.2. Các bước khởi chạy

**Bước 1 — Clone mã nguồn về máy chủ**

```bash
git clone https://github.com/hight-87ii/dtc245160042-HoangCongSu.git
cd dtc245160042-HoangCongSu
```

**Bước 2 — Tạo tệp biến môi trường `.env` tại thư mục gốc**

```bash
cat << 'EOF' > .env
MYSQL_ROOT_PASSWORD=SuperSecretRootPass123
MYSQL_DATABASE=recipe_db
MYSQL_USER=recipe_user
MYSQL_PASSWORD=RecipeUserPass123
EOF
```

> ⚠️ `.env` đã nằm trong `.gitignore` — không commit tệp này lên Git. Khi triển khai thực tế, hãy đổi toàn bộ mật khẩu sang chuỗi mạnh và lưu trong trình quản lý bí mật.

**Bước 3 — Kiểm tra cú pháp và khởi động hệ thống**

```bash
# Kiểm tra cú pháp tệp compose (đồng thời xác nhận đã đọc được .env)
docker compose config

# Khởi chạy toàn bộ hệ thống ở chế độ nền
docker compose up -d --build
```

**Bước 4 — Kiểm tra trạng thái các container**

```bash
docker compose ps
```

Kết quả mong đợi: **11 container** đều ở trạng thái `Up` / `Running`:

```
recipe-nginx   recipe-wordpress   recipe-mysql       recipe-phpmyadmin
recipe-prometheus   recipe-grafana   recipe-node-exporter   recipe-cadvisor
recipe-mysql-exporter   recipe-loki   recipe-promtail
```

**Bước 5 — Hoàn tất cài đặt WordPress**

Truy cập `http://<IP_MAY_CHU>/wp-admin`, chọn ngôn ngữ, tạo tài khoản quản trị và đăng nhập. Sau đó cài đặt giao diện/tạo bài viết cho các thực đơn: **món ăn**, **nguyên liệu**, **danh mục**.

---

## 🌐 4. Danh sách cổng dịch vụ & đường dẫn truy cập

Thay `localhost` bằng địa chỉ IP máy chủ Ubuntu nếu truy cập từ máy khác.

| Dịch vụ | URL | Ghi chú |
| --- | --- | --- |
| **Website RecipeHub (qua Nginx)** | `http://localhost` | Cổng `80` — giao diện người dùng xem công thức, nguyên liệu, danh mục. |
| **Quản trị WordPress** | `http://localhost/wp-admin` | Tài khoản tạo trong bước cài đặt WordPress. |
| **phpMyAdmin** | `http://localhost:8081` | Host: `db` • User: `recipe_user` • Pass: `RecipeUserPass123` |
| **Grafana** | `http://localhost:3000` | User/Pass mặc định: `admin` / `admin` (đổi ngay sau lần đăng nhập đầu). |
| **Prometheus** | `http://localhost:9090` | Xem trạng thái scrape tại **Status → Targets**. |
| **Loki API** | `http://localhost:3100` | Data source nội bộ cho Grafana: `http://loki:3100` |

> 🔒 MySQL (`3306`) và WordPress (`80` nội bộ) **không** publish port ra host — chỉ truy cập được từ trong mạng Docker.

---

## 📊 5. Giám sát với Grafana / Prometheus và truy vấn log bằng LogQL

### 5.1. Thiết lập Prometheus + Grafana

1. Truy cập `http://localhost:3000`, đăng nhập bằng `admin` / `admin`.
2. **Connections → Data sources → Add new data source → Prometheus**.
3. Nhập URL nội bộ `http://prometheus:9090` → **Save & test**.
4. **Dashboards → New → Import**, nhập các Dashboard ID:

| Dashboard | ID | Nội dung giám sát |
| --- | --- | --- |
| Node Exporter Full | `1860` | CPU, RAM, disk, network của máy chủ Linux |
| Docker / cAdvisor | `14282` (hoặc `193`) | Tài nguyên từng container |
| MySQL Overview | `7362` | Truy vấn, kết nối, bộ đệm của MySQL |

Các job scrape đã khai báo trong `prometheus/prometheus.yml`:

| Job | Target |
| --- | --- |
| `prometheus` | `localhost:9090` |
| `node-exporter` | `node-exporter:9100` |
| `cadvisor` | `cadvisor:8080` |
| `mysql-exporter` | `mysql-exporter:9104` |

### 5.2. Tích hợp Loki và truy vấn log

1. **Connections → Data sources → Add new data source → Loki**.
2. Nhập URL nội bộ `http://loki:3100` → **Save & test**.
3. Vào **Explore**, chọn nguồn **Loki** và chạy các truy vấn:

**Query 1 — Toàn bộ log của container WordPress:**

```logql
{container="recipe-wordpress"}
```

**Query 2 — Lọc các dòng log chứa từ khóa lỗi trên WordPress hoặc MySQL:**

```logql
{container=~"recipe-wordpress|recipe-mysql"} |= "error"
```

**Query 3 — Log truy cập của Nginx theo nhãn `service`:**

```logql
{service="nginx"}
```

> Nhãn `container` và `service` được tạo bởi `relabel_configs` trong `promtail/promtail-config.yml` (dựa trên `__meta_docker_container_name` và label `com.docker.compose.service`).

---

## 🔒 6. Các biện pháp bảo mật (System Hardening) đã áp dụng

1. **Cô lập mạng nội bộ (Network Isolation)**
   - Chia hệ thống thành 3 bridge network riêng biệt: `frontend_net`, `backend_net`, `monitoring_net`.
   - `recipe-mysql` chỉ nằm trong `backend_net` và **không publish cổng 3306** ra host → ngăn quét cổng và tấn công trực tiếp từ bên ngoài.

2. **Tách thông tin nhạy cảm ra `.env`**
   - Mật khẩu DB không hardcode trong `docker-compose.yml`, được nạp qua `env_file` + biến `${...}`.
   - `.env` nằm trong `.gitignore`, không bị đẩy lên GitHub.

3. **Nguyên tắc đặc quyền tối thiểu (PoLP)**
   - WordPress và MySQL Exporter kết nối DB bằng tài khoản riêng `recipe_user` (chỉ có quyền trên `recipe_db`) thay vì `root`.

4. **HTTP Security Headers tại Nginx** (`nginx/default.conf`)
   - `X-Frame-Options "SAMEORIGIN" always;` — chống Clickjacking.
   - `X-XSS-Protection "1; mode=block" always;` — giảm thiểu Cross-Site Scripting.
   - `X-Content-Type-Options "nosniff" always;` — chống MIME-sniffing.

5. **Mount chỉ đọc & hạn chế đặc quyền**
   - `nginx/default.conf` mount với `:ro`.
   - cAdvisor và Promtail mount `/`, `/var/run`, `/sys`, `/var/lib/docker`, `/var/lib/docker/containers`, `docker.sock` ở chế độ chỉ đọc → giảm nguy cơ leo thang đặc quyền / sửa tệp hệ thống từ trong container.
   - Không container nào chạy `privileged: true`.

6. **Dữ liệu bền vững & tách biệt**
   - 5 named volumes (`db_data`, `wp_data`, `prometheus_data`, `grafana_data`, `loki_data`) giúp dữ liệu tồn tại độc lập với vòng đời container.

---

## ⚡ 7. Lệnh vận hành & quản trị nhanh

```bash
# Xem log trực tiếp của một dịch vụ
docker compose logs -f nginx
docker compose logs -f wordpress

# Kiểm tra HTTP Security Headers trả về từ Nginx
curl -I http://localhost

# Kiểm tra tài nguyên tiêu thụ của các container
docker stats

# Khởi động lại toàn bộ hệ thống
docker compose restart

# Dừng hệ thống (giữ nguyên dữ liệu trong volumes)
docker compose down

# Dừng và xoá toàn bộ dữ liệu (volumes) — chỉ dùng khi muốn reset
docker compose down -v
```

---

## 🧯 8. Xử lý sự cố thường gặp

| Hiện tượng | Nguyên nhân / Cách xử lý |
| --- | --- |
| `docker compose config` báo thiếu biến | Chưa tạo `.env` hoặc sai thư mục chạy lệnh. |
| WordPress báo *Error establishing a database connection* | MySQL chưa sẵn sàng hoặc sai `MYSQL_*` trong `.env`. Xem `docker compose logs db`. Chờ `db` healthy rồi `docker compose restart wordpress`. |
| Grafana không có dữ liệu | Kiểm tra **Prometheus → Status → Targets**; data source phải là `http://prometheus:9090` (không dùng `localhost`). |
| Loki trả về lỗi `400` / không thấy log | Promtail cần quyền đọc `/var/run/docker.sock`; kiểm tra `docker compose logs promtail`. |
| Cổng `80` đã bị chiếm | Tắt dịch vụ đang dùng cổng 80 (`sudo systemctl stop apache2` / `nginx`) hoặc đổi mapping trong `docker-compose.yml`. |
| Grafana/Prometheus không truy cập được từ máy khác | Mở firewall: `sudo ufw allow 80,3000,3100,8081,9090/tcp`. |

---

## 📄 9. Ghi chú

- Toàn bộ image sử dụng tag `latest`/`8.0` nhằm mục đích học tập; môi trường production nên pin phiên bản cụ thể để đảm bảo tính tái lập.
- Tệp `.env` mẫu được mô tả tại **Bước 2**. Không commit mật khẩu thật lên repository.

---

<p align="center">
  <b>Hoàng Công Sự — DTC245160042</b><br>
  Triển khai và Quản trị Hệ thống Phần mềm — ICTU
</p>
