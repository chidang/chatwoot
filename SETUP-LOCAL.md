# Chatwoot — Local Dev Setup (macOS)

> File này tóm tắt các bước còn lại bạn cần chạy **trên Mac**. Phần clone repo và file `.env` đã được tạo sẵn.

## Trạng thái hiện tại

- ✅ Repo `chidang/chatwoot` đã clone vào folder này
- ✅ `upstream` remote → `chatwoot/chatwoot`
- ✅ `.env` đã tạo (PostgreSQL/Redis trỏ `localhost`, `SECRET_KEY_BASE` đã sinh)
- ✅ `.env` đã có trong `.gitignore`

## ⚠️ Lưu ý phiên bản (KHÁC với prompt setup)

File `.ruby-version` của repo yêu cầu **Ruby 3.4.4** (không phải 3.2.x), `.nvmrc` yêu cầu **Node 24.13.0** (không phải 20.x). Phải dùng đúng version trong repo.

| Dependency | Version yêu cầu | Quản lý bằng |
|---|---|---|
| Ruby | **3.4.4** | rbenv |
| Node | **24.13.0** | nvm |
| Yarn | latest | corepack/npm |
| PostgreSQL | 14.x (đã cài Homebrew) | brew |
| Redis | latest (đã cài Homebrew) | brew |
| ImageMagick | latest | brew |

---

## Bước 1 — Kiểm tra và cài dependencies (chạy trên Mac)

```bash
# Đảm bảo Homebrew sẵn sàng
brew --version

# PostgreSQL & Redis (đã có sẵn — chỉ cần đảm bảo đang chạy)
brew services start postgresql@14
brew services start redis
brew services list                  # cả hai phải "started"

# ImageMagick (nếu chưa có)
brew install imagemagick

# rbenv + Ruby 3.4.4
brew install rbenv ruby-build
rbenv install 3.4.4                 # bỏ qua nếu đã cài
rbenv global 3.4.4
ruby -v                             # phải in "ruby 3.4.4"

# nvm + Node 24.13.0
brew install nvm
mkdir -p ~/.nvm
# Thêm vào ~/.zshrc nếu chưa có:
#   export NVM_DIR="$HOME/.nvm"
#   [ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"
nvm install 24.13.0
nvm use 24.13.0
node -v                             # phải in "v24.13.0"

# Yarn (Chatwoot dùng pnpm/yarn classic — kiểm tra package.json để chắc)
corepack enable
corepack prepare yarn@stable --activate
yarn -v
```

> Nếu `rbenv install 3.4.4` báo lỗi compile trên Apple Silicon, chạy:
> ```bash
> brew install openssl@3 readline libyaml gmp
> RUBY_CONFIGURE_OPTS="--with-openssl-dir=$(brew --prefix openssl@3)" \
>   rbenv install 3.4.4
> ```

---

## Bước 2 — Tạo PostgreSQL user + database

```bash
cd ~/Project/startup/chatwoot

psql postgres -c "CREATE USER chatwoot WITH PASSWORD 'chatwoot' CREATEDB SUPERUSER;"
psql postgres -c "CREATE DATABASE chatwoot_dev OWNER chatwoot;"

# Kiểm tra:
psql -U chatwoot -d chatwoot_dev -c "SELECT version();"
```

> User cần `CREATEDB` để chạy được test database; `SUPERUSER` cần thiết để Chatwoot tạo extension `pg_trgm`, `pgcrypto`. Nếu lo bảo mật thì sau khi setup xong có thể `ALTER USER chatwoot NOSUPERUSER`.

---

## Bước 3 — Cài gem và package frontend

```bash
cd ~/Project/startup/chatwoot

# Đảm bảo dùng đúng Ruby/Node version (rbenv/nvm tự đọc .ruby-version / .nvmrc)
ruby -v        # 3.4.4
node -v        # 24.13.0

# Bundler
gem install bundler
bundle install                    # ~3-5 phút

# Frontend deps
yarn install                      # ~2-3 phút

# (Nếu repo dùng pnpm — kiểm tra packageManager trong package.json)
# pnpm install
```

---

## Bước 4 — Khởi tạo database

```bash
bundle exec rails db:chatwoot_prepare
```

Lệnh này sẽ:
- Tạo schema (`db:create db:schema:load`)
- Chạy migrations
- Seed data (admin user, account mẫu)

Mặc định admin: `john@acme.inc` / `Password1!` (xem `db/seeds.rb` để chắc).

---

## Bước 5 — Chạy app (3 terminal)

```bash
# Terminal 1 — Rails API
bundle exec rails server
# → http://localhost:3000

# Terminal 2 — Vite hot-reload (Chatwoot mới dùng Vite, không phải webpack)
bin/vite dev
# (hoặc: yarn dev — nếu có script tương ứng trong package.json)

# Terminal 3 — Sidekiq background jobs
bundle exec sidekiq -C config/sidekiq.yml
```

> Bạn cũng có thể dùng `overmind` hoặc `foreman` để chạy cả 3 cùng lúc:
> ```bash
> brew install overmind
> overmind start -f Procfile.dev
> ```

---

## Kết quả mong đợi

- Truy cập http://localhost:3000 → trang đăng nhập Chatwoot
- Login với account seed → vào dashboard
- Sửa file Vue/JS trong `app/javascript/` → hot-reload tự động
- Sidekiq logs hiển thị job processing

---

## Lệnh hay dùng khi dev

```bash
# Migration
bundle exec rails generate migration AddColumnToTable column:type
bundle exec rails db:migrate

# Rails console
bundle exec rails console

# Routes
bundle exec rails routes | grep keyword

# Test
bundle exec rspec
yarn test                         # frontend test (nếu có)

# Reset toàn bộ database
bundle exec rails db:drop db:create db:chatwoot_prepare

# Cập nhật từ upstream
git fetch upstream
git merge upstream/develop        # hoặc upstream/main tuỳ branch
```

---

## Troubleshoot nhanh

| Lỗi | Cách xử lý |
|---|---|
| `pg_config not found` khi `bundle install` | `brew install libpq && bundle config build.pg --with-pg-config=$(brew --prefix libpq)/bin/pg_config` |
| `nokogiri` build lỗi | `bundle config build.nokogiri --use-system-libraries` |
| `redis connection refused` | `brew services restart redis` |
| `pg connection refused` | `brew services restart postgresql@14` |
| Vite port 3036 conflict | `lsof -ti:3036 \| xargs kill` |
| `chatwoot_dev` đã tồn tại | `psql postgres -c "DROP DATABASE chatwoot_dev;"` rồi chạy lại Bước 2 |

---

## Sau khi setup OK

Bạn có thể xoá file này và `chatwoot-setup-prompt.md` nếu muốn (chúng đã trong `.gitignore` mặc định? — kiểm tra trước khi commit).
