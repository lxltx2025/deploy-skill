# Nginx 反向代理 + HTTPS(certbot)

## 架构要点(必读)

宿主机 Nginx **只透传到前端容器 `127.0.0.1:8080`**。前端容器内置的 nginx 已处理 `/api` 和 `/socket.io`,宿主机**不要重复配 `/api`**,否则路由打架导致 404/502。

## 安装

```bash
sudo apt install nginx certbot python3-certbot-nginx -y
```

## 站点配置

```bash
sudo tee /etc/nginx/sites-available/<domain> >/dev/null <<'EOF'
upstream app_frontend {
    server 127.0.0.1:8080;
    keepalive 32;
}
server {
    listen 80;
    listen [::]:80;
    server_name <domain> www.<domain>;

    # certbot 验证目录(申请证书前必须能访问)
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        proxy_pass http://app_frontend;
        proxy_http_version 1.1;

        # websocket 支持(socket.io)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 长连接(实时推送不超时)
        proxy_read_timeout 86400s;
    }

    client_max_body_size 50m;
}
EOF

sudo ln -sf /etc/nginx/sites-available/<domain> /etc/nginx/sites-enabled/<domain>
sudo rm -f /etc/nginx/sites-enabled/default   # 删默认站,避免抢 server_name
sudo nginx -t && sudo systemctl reload nginx
```

## 申请 HTTPS 证书(非交互)

```bash
sudo certbot --nginx \
  -d <domain> -d www.<domain> \
  --non-interactive --agree-tos --no-eff-email \
  -m <letsencrypt_email> --redirect
```
- `--redirect`:certbot 自动加 HTTP→HTTPS 301 跳转
- 证书路径 `/etc/letsencrypt/live/<domain>/`,自动续期 `certbot.timer` 默认已启用
- **前提**:DNS 已解析到本机 + 80 端口公网可达(腾讯云安全组放行 80/443)

## 常见 nginx 报错

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| `nginx: [emerg] duplicate location "/api"` | 宿主和容器都配了 /api | 删宿主机的 /api location,只透传 8080 |
| 502 Bad Gateway | 前端容器没起 / 端口不对 | `docker compose ps` + `curl 127.0.0.1:8080` 先确认容器 |
| 证书申请失败 connection refused | 80 端口没放行 / DNS 没解析 | 安全组放 80 + `dig` 确认解析 |
| www 跳转死循环 | certbot redirect 写错 | 检查 server_name 含 www,return 301 指向 https |
