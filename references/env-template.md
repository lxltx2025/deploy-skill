# .env 模板与密钥生成

## 生成强随机密钥(不要硬编码)

```bash
PG_PWD=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)
JWT_ACCESS=$(openssl rand -hex 32)
JWT_REFRESH=$(openssl rand -hex 32)
ADMIN_PASS=$(openssl rand -base64 18 | tr -d '/+=' | head -c 20)
```

> JWT 用两套(access/refresh)做双令牌;access 短(15m)、refresh 长(7d)。

## 完整 .env 模板

把 `<domain>` `<smtp_*>` 等占位符替换成实际值;变量名要与后端代码读取的一致(下面是 NestJS+Prisma 常见命名,确认你的项目代码)。

```bash
sudo tee /opt/<app>/.env >/dev/null <<EOF
# --- Postgres ---
POSTGRES_USER=staff
POSTGRES_PASSWORD=$PG_PWD
POSTGRES_DB=staff_man
POSTGRES_PORT=5432
DATABASE_URL=postgresql://staff:$PG_PWD@postgres:5432/staff_man?schema=public

# --- JWT 双令牌 ---
JWT_ACCESS_SECRET=$JWT_ACCESS
JWT_REFRESH_SECRET=$JWT_REFRESH
JWT_ACCESS_EXPIRES=15m
JWT_REFRESH_EXPIRES=7d

# --- Web ---
CORS_ORIGIN=https://<domain>
VITE_API_BASE_URL=/api
SWAGGER_ENABLED=false

# --- SMTP(验证码登录,授权码非登录密码) ---
SMTP_HOST=smtp.163.com
SMTP_PORT=465
SMTP_USER=<smtp_user>
SMTP_PASS=<smtp_授权码>
SMTP_FROM_NAME=<发件人名>
SMTP_SECURE=true

# --- 种子账号(首次启动用,之后改 DB 不改这里) ---
SEED_ADMIN_USERNAME=<admin_user>
SEED_ADMIN_PASSWORD=$ADMIN_PASS
EOF
sudo chown deploy:deploy /opt/<app>/.env && sudo chmod 600 /opt/<app>/.env
```

## SMTP 授权码说明

163/QQ/企业邮箱的 SMTP 不能用登录密码,要在邮箱后台开 SMTP 服务并取**授权码**。
- 163: 设置 → POP3/SMTP → 开启 → 取授权码
- QQ: 设置 → 账户 → 开启 SMTP → 取授权码
- `SMTP_SECURE=true` 配 465 端口(SSL);若用 587 端口则 `SMTP_SECURE=false`(STARTTLS)

## docker-compose 端口绑定(阶段 6)

容器端口全部加 `127.0.0.1:` 前缀,不裸暴露:
```yaml
services:
  postgres:
    ports: ["127.0.0.1:5432:5432"]
  backend:
    ports: ["127.0.0.1:3000:3000"]
    restart: unless-stopped
  frontend:
    ports: ["127.0.0.1:8080:80"]
    restart: unless-stopped
```
