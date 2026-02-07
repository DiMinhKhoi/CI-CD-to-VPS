# Hướng Dẫn Cài Đặt CI/CD

Tài liệu này hướng dẫn thiết lập hệ thống CI/CD tự động deploy ứng dụng từ GitHub lên VPS sử dụng Docker và GitHub Actions Self-hosted Runner.

## 📋 Quy Ước Biến

| Placeholder | Mô tả | Ví dụ |
|-------------|-------|-------|
| `<your-username>` | Username GitHub của bạn | `diminhkhoi` |
| `<your-project>` | Tên project/thư mục deploy | `myproject` |
| `<your-service>` | Tên service trong docker-compose | `my-web` |
| `<deploy-repo>` | Tên repo chứa config deploy | `CI-CD-to-VPS` |
| `<source-repo>` | Tên repo chứa source code | `MyApp` |

---

## 📋 Tổng Quan Kiến Trúc

```
┌─────────────────────┐      ┌──────────────────────┐      ┌─────────────┐
│  <source-repo>      │      │  <deploy-repo>       │      │    VPS      │
│  (Source Code)      │      │  (Deploy Config)     │      │             │
├─────────────────────┤      ├──────────────────────┤      ├─────────────┤
│ 1. Push code/tag    │      │                      │      │             │
│ 2. Build Docker     │      │                      │      │             │
│ 3. Push to ghcr.io  │─────►│ 4. Nhận trigger      │─────►│ 5. Pull     │
│ 4. Trigger dispatch │      │ 5. Update .env       │      │ 6. Run      │
└─────────────────────┘      │ 6. docker compose up │      │             │
                             └──────────────────────┘      └─────────────┘
```

---

## 🔧 Phần 1: Cấu Hình VPS

### 1.1 Cài đặt Docker
```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

### 1.2 Tạo User cho Runner
```bash
# Tạo user
sudo adduser runner

# Thêm vào group docker
sudo usermod -aG docker runner

# Thêm quyền sudo (tùy chọn)
echo "runner ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/runner
```

### 1.3 Tạo Thư Mục Deploy
```bash
sudo mkdir -p /opt/<your-project>
sudo chown -R runner:runner /opt/<your-project>
```

### 1.4 Cài đặt GitHub Actions Runner
```bash
# Chuyển sang user runner
su - runner

# Tải runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Cấu hình (lấy token từ GitHub: Settings > Actions > Runners > New self-hosted runner)
./config.sh --url https://github.com/<your-username>/<deploy-repo> --token YOUR_TOKEN --work _work

# Cài đặt như service (chạy với root)
exit
cd /home/runner/actions-runner
sudo ./svc.sh install runner
sudo ./svc.sh start
```

### 1.5 Cài đặt công cụ cần thiết
```bash
sudo apt install rsync curl -y
```

---

## 🔐 Phần 2: Tạo GitHub Secrets

### 2.1 Tạo Personal Access Token
1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Chọn scopes:
   - `repo` (để trigger workflow)
   - `read:packages` (để pull private images)
   - `write:packages` (để push images)
4. Copy token

### 2.2 Thêm Secrets vào Repo

**Repo `<deploy-repo>`:**
| Secret Name | Mô tả |
|-------------|-------|
| `DEPLOY_REPO_TOKEN` | Token để login ghcr.io và trigger workflow |

**Repo `<source-repo>`:**
| Secret Name | Mô tả |
|-------------|-------|
| `DEPLOY_TOKEN` | Token để trigger deploy workflow |

---

## 📁 Phần 3: Cấu Hình Repo Deploy

### 3.1 Cấu trúc thư mục
```
<deploy-repo>/
├── .github/
│   └── workflows/
│       └── deploy.yml      # Workflow deploy
├── docker-compose.yml      # Cấu hình Docker
├── .env.example            # Mẫu biến môi trường
├── .gitignore
└── README.md
```

### 3.2 File docker-compose.yml
```yaml
services:
  <your-service>:
    image: ghcr.io/<your-username>/<your-service>:${<YOUR_SERVICE>_TAG}
    container_name: <your-service>
    ports:
      - "3001:80"
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 3.3 File deploy.yml (tóm tắt)
```yaml
name: Deploy on VPS (self-hosted)

on:
  repository_dispatch:
    types: [deploy]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sync files to VPS
        run: rsync -a --delete --exclude ".env" ./ /opt/<your-project>/
      
      - name: Update .env with tag
        run: |
          # Ghi tag từ payload vào .env
          # <YOUR_SERVICE>_TAG=1.0.0
      
      - name: Login to ghcr.io
        run: echo "${{ secrets.DEPLOY_REPO_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
      
      - name: Pull & Up
        run: |
          docker compose pull
          docker compose up -d --remove-orphans
```

---

## 🚀 Phần 4: Cấu Hình Repo Source (Build)

### 4.1 File build-and-deploy.yml
```yaml
name: Build and Deploy to Production

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: <your-username>/<your-service>

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Login to ghcr.io
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        run: |
          TAG="${GITHUB_REF_NAME}"
          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}"
          docker build -t $IMAGE:$TAG -t $IMAGE:latest .
          docker push $IMAGE:$TAG
          docker push $IMAGE:latest

      - name: Trigger Deploy
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" \
            https://api.github.com/repos/<your-username>/<deploy-repo>/dispatches \
            -d '{"event_type":"deploy","client_payload":{"service":"<your-service>","tag":"'"${GITHUB_REF_NAME}"'"}}'
```

---

## 🎯 Phần 5: Quy Trình Release

### 5.1 Release phiên bản mới
```bash
# 1. Commit code
git add .
git commit -m "feat: new feature"

# 2. Tạo tag
git tag v1.0.0

# 3. Push tag (trigger CI/CD tự động)
git push origin v1.0.0
```

### 5.2 Kiểm tra trạng thái
```bash
# Trên VPS
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

---

## 🔍 Phần 6: Debug & Troubleshooting

### 6.1 Xem log runner
```bash
# Log service
sudo journalctl -u actions.runner.<your-username>-<deploy-repo>.<runner-name> -f

# Log file
tail -f /home/runner/actions-runner/_diag/*.log
```

### 6.2 Lỗi thường gặp

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `rsync: command not found` | Chưa cài rsync | `sudo apt install rsync` |
| `Permission denied` | User runner không có quyền | `sudo chown -R runner:runner /opt/<your-project>` |
| `unauthorized` | Chưa login ghcr.io | Thêm step Login to ghcr.io |
| `repository name must be lowercase` | Tên image có chữ hoa | Đổi thành chữ thường |
| `Waiting for runner` | Runner chưa chạy | Chạy `sudo ./svc.sh start` |

---

## ✅ Checklist Hoàn Thành

- [ ] VPS đã cài Docker
- [ ] User runner đã tạo và có quyền docker
- [ ] Thư mục `/opt/<your-project>` đã tạo
- [ ] GitHub Runner đã cài và chạy như service
- [ ] Secrets đã thêm vào cả 2 repo
- [ ] File docker-compose.yml đúng registry
- [ ] File deploy.yml có step login ghcr.io
- [ ] Test release thành công
