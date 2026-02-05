# Tin Nghia Auto - CI/CD to VPS

Hệ thống điều phối triển khai tự động (Deployment Orchestrator) cho các dịch vụ của Tin Nghia Auto trên môi trường Production.

## 📌 Chức năng chính
- **Quản lý cấu hình Docker**: Định nghĩa hạ tầng chạy Production cho API và Web Frontend.
- **Tự động hóa triển khai**: Sử dụng GitHub Actions (Self-hosted Runner) để tự động cập nhật phiên bản mới nhất từ Production.
- **Quản lý phiên bản**: Điều phối việc cập nhật tags của các Docker images thông qua biến môi trường.

## 🚀 Kiến trúc triển khai
- **API**: Chạy trên cổng `7001` (ghcr.io/tinnghiaauto/admin-tinnghia-api).
- **Web**: Chạy trên cổng `3001` (ghcr.io/tinnghiaauto/admin-tinnghia-web).
- **Môi trường**: Sử dụng Docker Compose để quản lý container đồng nhất.

## 🛠 Cách sử dụng

### 1. Chuẩn bị môi trường trên VPS
Đảm bảo đã cài đặt Docker và Docker Compose. Thư mục triển khai mặc định: `/opt/tinnghiaauto`.

### 2. File cấu hình `.env`
Tạo file `.env` dựa trên `.env.example`:
```env
ADMIN_TINNGHIA_WEB_TAG=latest
ADMIN_TINNGHIA_API_TAG=latest
```

### 3. Quy trình CD tự động
Quy trình này được kích hoạt tự động thông qua `repository_dispatch` từ các repo mã nguồn chính. Bạn không cần thao tác thủ công trừ khi cấu hình hạ tầng thay đổi.

## 🔒 Bảo mật & Bảo trì
- Repo này **không** nên chứa bất kỳ thông tin nhạy cảm nào (mật khẩu database, API keys). Tất cả nên nằm trong file `.env` trên server.
- Định kỳ hệ thống sẽ tự động dọn dẹp các images Docker cũ để tiết kiệm dung lượng ổ cứng.