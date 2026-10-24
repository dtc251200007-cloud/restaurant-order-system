# Đề 12 — Hệ thống Quản lý Nhà hàng / Order

Môn: Triển khai và Quản trị Hệ thống Phần mềm. Sinh viên: **Lê Văn Quyến** — **DTC251200007**.

## Yêu cầu
- Ubuntu 22.04+ hoặc Windows với Docker Desktop; Docker Engine và plugin `docker compose`.
- Các cổng 8080, 5050, 3001, 9090, 3100 chưa bị ứng dụng khác chiếm.

## Khởi động
```bash
cp .env.example .env
nano .env
# THAY TẤT CẢ chuỗi CHANGE_ME bằng mật khẩu/bí mật đủ mạnh. Không commit .env.
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

Các địa chỉ từ CHÍNH máy đang chạy Docker:

| Thành phần | URL | Tài khoản |
|---|---|---|
| Website qua Nginx | http://localhost:8080 | dtc251200007@ictu.edu.vn / quyentn123456789 trong .env |
| pgAdmin | http://127.0.0.1:5051 | dtc251200007@ictu.edu.vn / quyentn123456789 |
| Grafana | http://localhost:3001 | admin / quyentn123456789 |
| Prometheus | http://localhost:9090 | không có đăng nhập, chỉ bind localhost |
| Loki HTTP API | http://localhost:3100/ready | chỉ bind localhost |

**pgAdmin > Add New Server**: Name `Restaurant`; tab Connection: Host `db`, Port `5432`, Maintenance database `restaurant` (hoặc POSTGRES_DB), Username / Password lấy trong `.env`. Bật Save password chỉ trên máy thực hành tin cậy.

**Grafana**: Dashboards > Restaurant > Restaurant Order | Monitoring. Explore > đổi datasource thành Loki. Các truy vấn mẫu LogQL:
```logql
{job="restaurant_app"}
{job="restaurant_app"} |= "request"
{job="restaurant_app"} |= "error"
{job="nginx"}
```
Truy vấn PromQL mẫu:
```promql
up
sum(rate(restaurant_http_requests_total[5m]))
pg_up
nginx_up
```

## Demo 7 bước
1. Truy cập website, đăng nhập bằng ADMIN_EMAIL và ADMIN_PASSWORD.
2. Tại **Thực đơn**, thêm món, sửa giá, chuyển ngừng bán.
3. Vào **Bàn ăn** > chọn bàn trống > chọn món và số lượng > **Xác nhận tạo đơn**.
4. Vào **Đơn hàng**, xem số tiền, **Thanh toán** hoặc **Hủy**; kiểm tra bàn trở lại trống.
5. Mở pgAdmin, xem các bảng `menu_items`, `dining_tables`, `orders`, `order_items` và số bản ghi.
6. Prometheus Targets: http://localhost:9090/targets; Grafana: dashboard đã provision, tạo traffic trên website để xem biểu đồ.
7. Grafana Explore > Loki > chạy ít nhất 3 truy vấn LogQL; chụp minh chứng.

## GitHub — tối thiểu 3 commit có ý nghĩa
Tạo repo mới với tên gắn mã SV, ví dụ `DTC251200007-restaurant-order` và commit theo 3 mốc THỰC SỰ thực hiện:
```bash
git init
git add app compose.yaml .env.example .gitignore README.md
git commit -m "feat: restaurant web with PostgreSQL and Docker Compose"
git add infra/nginx
git commit -m "infra: configure Nginx reverse proxy and security headers"
git add infra/prometheus infra/grafana infra/loki infra/promtail
git commit -m "observability: add Prometheus Grafana Loki Promtail"
# Sau đó tự thêm GitHub remote rồi push theo repo của bạn.
```
Lưu ý: commit 1 phía trên đã include file compose cuối cùng và README, nên để lịch sử thể hiện rõ các mốc phát triển hơn, tốt nhất thực hiện theo từng giai đoạn riêng trong quá trình làm bài.

## Lệnh kiểm tra và khắc phục
```bash
docker compose ps
docker compose logs --tail=100 app
docker compose logs --tail=100 nginx
docker compose logs --tail=100 loki promtail
curl -i http://localhost:8080/api/health
curl -s http://localhost:9090/api/v1/targets
# Dừng mà giữ dữ liệu:
docker compose down
# XÓA SẠCH toàn bộ dữ liệu, chỉ chạy khi thật sự muốn:
# docker compose down -v
```

## Cấu trúc và bảo mật
- `app`: Node.js Express, cookie HttpOnly SameSite Strict được ký HMAC, bcrypt password, truy vấn SQL có parameter, app user non-root và read-only filesystem; metrics ở `/metrics` chỉ accessible qua mạng Docker monitor trực tiếp (Nginx không nên public đường này).
- `db`: PostgreSQL 16 trong private backend network; không publish port DB ra bên ngoài.
- `nginx`: reverse proxy, security headers và log ra volume.
- `prometheus`, `postgres_exporter`, `nginx_exporter`: thu thập metrics app, DB và Nginx.
- `grafana`: datasource Prometheus và Loki, dashboard tạo sẵn.
- `loki`, `promtail`: log tập trung, đọc app/nginx log qua volumes.
- Có tách `frontend`, `backend` (internal), `monitor`, restart policy và hạn chế quyền; secret được cấu hình bằng `.env` chưa commit.

### Giới hạn cần lưu ý
- **Môi trường lab nội bộ**: Nginx đang dùng HTTP với security headers, đáp ứng phương án yêu cầu đề bài. Nếu mở Internet, cấu hình TLS thực, cookie Secure=true, hạn chế truy cập trang quản trị; không expose 8080 bừa bãi.
- Dashboard có graph ứng dụng, Nginx, PostgreSQL; **không gồm cAdvisor / per-container CPU** trong bản này. Nếu cần chỉ số tài nguyên từng container, tích hợp cAdvisor hoặc exporter phù hợp theo môi trường.
- Promtail đã EOL từ 02/03/2026, giữ vì đề bài yêu cầu đích danh; hệ thống production nên chuyển Grafana Alloy.
- Token đăng nhập hết hạn sau 8 giờ; không có chức năng phân quyền nâng cao hay thanh toán điện tử thật.
- Mật khẩu ADMIN trong `.env` được cập nhật lại trong database ở mỗi lần khởi động app.
- Dữ liệu seed chỉ thêm khi bảng danh mục/bàn còn trống, không ghi đè dữ liệu đã có.

## Sơ đồ luồng
Browser → Nginx → Node.js Express → PostgreSQL; pgAdmin → PostgreSQL;
Prometheus → app `/metrics` + PostgreSQL exporter + Nginx exporter → Grafana;
Node.js/Nginx logfile → Promtail → Loki → Grafana Explore.
