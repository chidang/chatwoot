# Prompt: Setup Chatwoot Development & Production Deployment

## Thông tin dự án

- **Source code**: `https://github.com/chidang/chatwoot`
- **Stack**: Ruby on Rails (backend) + Vue.js (frontend)
- **Dev machine**: macOS (Apple Silicon)
- **Production**: Ubuntu Server + Docker

---

## Nguyên tắc môi trường

| | Local (Dev) | Production (Ubuntu) |
|---|---|---|
| PostgreSQL | ✅ Native (đã cài trên Mac) | ✅ Docker container |
| Redis | ✅ Native (đã cài trên Mac) | ✅ Docker container |
| Rails + Vue | ✅ Chạy thẳng trên máy | ✅ Docker container |
| Docker | ❌ Không dùng | ✅ Toàn bộ stack |

---

## PHẦN 1: Local Development (macOS)

### Yêu cầu

Kiểm tra và cài đặt nếu chưa có:

| Dependency | Version | Ghi chú |
|---|---|---|
| Ruby | 3.2.x | Dùng `rbenv` để quản lý |
| Node.js | 20.x | Dùng `nvm` để quản lý |
| Yarn | latest | Package manager frontend |
| PostgreSQL | 14.x | Đã cài native trên Mac, port 5432 |
| Redis | latest | Đã cài native trên Mac, port 6379 |
| ImageMagick | latest | Xử lý ảnh |

> PostgreSQL và Redis đã có sẵn trên máy, **không cài thêm Docker** cho local.

### Các bước thực hiện

**1. Clone repository**
```bash
git clone git@github.com:chidang/chatwoot.git
cd chatwoot
git remote add upstream https://github.com/chatwoot/chatwoot.git
```

**2. Tạo PostgreSQL user + database cho Chatwoot**
```bash
psql postgres -c "CREATE USER chatwoot WITH PASSWORD 'chatwoot';"
psql postgres -c "CREATE DATABASE chatwoot_dev OWNER chatwoot;"
```

**3. Cấu hình `.env`**

Copy từ `.env.example`:
```bash
cp .env.example .env
```

Sửa các giá trị sau trong `.env`:
```env
# Database — trỏ vào PostgreSQL native trên Mac
POSTGRES_HOST=localhost
POSTGRES_USERNAME=chatwoot
POSTGRES_PASSWORD=chatwoot
DATABASE_URL=postgresql://chatwoot:chatwoot@localhost:5432/chatwoot_dev

# Redis — trỏ vào Redis native trên Mac
REDIS_URL=redis://localhost:6379

# App
SECRET_KEY_BASE=<chạy: openssl rand -hex 64>
FRONTEND_URL=http://localhost:3000
```

**4. Cài dependencies**
```bash
bundle install
yarn install
```

**5. Khởi tạo database**
```bash
bundle exec rails db:chatwoot_prepare
```

**6. Chạy app — mở 3 terminal**

```bash
# Terminal 1: Rails API
bundle exec rails server
# → http://localhost:3000

# Terminal 2: Vue.js hot-reload
yarn dev

# Terminal 3: Background jobs
bundle exec sidekiq
```

### Kết quả mong đợi
- Truy cập `http://localhost:3000` thành công
- Hot-reload hoạt động khi sửa file Vue.js
- Không cần Docker chạy

### Lệnh hay dùng khi dev

```bash
# Tạo migration mới
bundle exec rails generate migration AddColumnToTable column:type

# Chạy migration
bundle exec rails db:migrate

# Rails console
bundle exec rails console

# Xem routes
bundle exec rails routes | grep keyword

# Chạy test
bundle exec rspec

# Reset database
bundle exec rails db:drop db:create db:chatwoot_prepare
```

---

## PHẦN 2: Production Deployment (Ubuntu + Docker)

### Yêu cầu server

| Yêu cầu | Chi tiết |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Docker | CE + Compose Plugin |
| Nginx | Reverse proxy + SSL |
| Domain | Đã trỏ về IP server |
| SSH Key | Đã setup cho GitHub Actions |

### Các file cần tạo trong repository

#### `Dockerfile.production`

```dockerfile
FROM ruby:3.2.2-slim

RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    nodejs \
    yarn \
    imagemagick \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY . .

RUN bundle exec rails assets:precompile

EXPOSE 3000
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

#### `docker-compose.production.yml`

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USERNAME}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: chatwoot_production
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    restart: always
    volumes:
      - redisdata:/data

  rails:
    image: chidang88/chatwoot:latest
    restart: always
    depends_on:
      - postgres
      - redis
    env_file: .env
    ports:
      - "127.0.0.1:3000:3000"
    command: bundle exec rails server -b 0.0.0.0

  sidekiq:
    image: chidang88/chatwoot:latest
    restart: always
    depends_on:
      - postgres
      - redis
    env_file: .env
    command: bundle exec sidekiq

volumes:
  pgdata:
  redisdata:
```

#### `.github/workflows/deploy.yml`

```yaml
name: Build & Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Login Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build & Push image
        run: |
          docker build -f Dockerfile.production \
            -t ${{ secrets.DOCKER_USERNAME }}/chatwoot:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/chatwoot:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: SSH vào server và deploy
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd ~/chatwoot
            docker compose -f docker-compose.production.yml pull
            docker compose -f docker-compose.production.yml up -d
            docker compose -f docker-compose.production.yml run --rm rails \
              bundle exec rails db:migrate
```

### GitHub Secrets cần thiết

Vào **GitHub repo → Settings → Secrets → Actions**, thêm:

```
DOCKER_USERNAME     ← Docker Hub username
DOCKER_PASSWORD     ← Docker Hub access token (không dùng password thật)
SERVER_HOST         ← IP Ubuntu server
SERVER_USER         ← user SSH (vd: ubuntu)
SERVER_SSH_KEY      ← nội dung private key SSH (-----BEGIN OPENSSH PRIVATE KEY-----)
```

### Setup Ubuntu Server lần đầu (chạy 1 lần duy nhất)

```bash
# 1. SSH vào server
ssh chidang@159.198.76.90
# Tôi sẽ nhập password
sudo su root

# 2. Cài Docker
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker

# 3. Tạo thư mục
mkdir ~/chatwoot && cd ~/chatwoot

# 4. Copy file lên server (chạy từ máy Mac)
# scp docker-compose.production.yml user@server-ip:~/chatwoot/
# scp .env.production user@server-ip:~/chatwoot/.env

# 5. Khởi tạo lần đầu
docker compose -f docker-compose.production.yml up -d
docker compose -f docker-compose.production.yml run --rm rails \
  bundle exec rails db:chatwoot_prepare

# 6. Cài Nginx + SSL
sudo apt install -y nginx certbot python3-certbot-nginx
```

### Cấu hình Nginx

```bash
sudo nano /etc/nginx/sites-available/chatwoot
```

```nginx
server {
    server_name yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/chatwoot /etc/nginx/sites-enabled/
sudo certbot --nginx -d yourdomain.com
sudo systemctl reload nginx
```

### `.env` trên Production server

```env
# Database — trỏ vào PostgreSQL container (tên service = postgres)
POSTGRES_HOST=postgres
POSTGRES_USERNAME=chatwoot
POSTGRES_PASSWORD=<mật khẩu mạnh>
DATABASE_URL=postgresql://chatwoot:<password>@postgres:5432/chatwoot_production

# Redis — trỏ vào Redis container (tên service = redis)
REDIS_URL=redis://redis:6379

# App
SECRET_KEY_BASE=<openssl rand -hex 64 — khác với local>
FRONTEND_URL=https://yourdomain.com

# SMTP
SMTP_ADDRESS=smtp.gmail.com
SMTP_USERNAME=your@gmail.com
SMTP_PASSWORD=your_app_password
MAILER_SENDER_EMAIL=your@gmail.com
```

> File `.env` production **KHÔNG commit lên Git** — quản lý thủ công trên server.

---

## PHẦN 3: Cấu trúc file sau khi setup

```
chatwoot/
├── .env                              # local dev (git ignore)
├── .env.example                      # mẫu (commit được)
├── Dockerfile.production             # build image production
├── docker-compose.production.yml     # chạy trên Ubuntu
└── .github/
    └── workflows/
        └── deploy.yml                # CI/CD tự động
```

**Không có** `docker-compose.dev.yml` — local dùng PostgreSQL + Redis native trên Mac.

---

## Workflow hàng ngày

```
1. Dev ở local
   └── sửa code → test tại localhost:3000
   └── PostgreSQL + Redis chạy native trên Mac (không cần Docker)

2. Commit & Push lên main
   └── git add .
   └── git commit -m "feat: mô tả tính năng"
   └── git push origin main

3. GitHub Actions tự động
   └── ✅ Build Docker image mới
   └── ✅ Push lên Docker Hub
   └── ✅ SSH vào Ubuntu → pull image → restart containers
   └── ✅ Chạy db:migrate tự động

4. Production cập nhật 🚀
```

---

## Hỏi thêm trước khi bắt đầu

Trước khi bắt đầu setup, hãy xác nhận với tôi:

1. Mac bạn dùng chip **Apple Silicon (M2)**?
2. PostgreSQL trên Mac cài bằng **Homebrew hay Postgres.app**? đây là cách chạy brew services stop postgresql@14
3. Ubuntu server đã có **domain** chưa, hay chỉ dùng IP tạm? chat.flexacommerce.com
4. Đã có tài khoản **Docker Hub** chưa? https://hub.docker.com/repositories/chidang88
