# 运维兜底(备份 / 续期 / 自愈)

## 数据库每日备份(保留 14 天)

```bash
sudo mkdir -p /opt/backups/postgres
sudo tee /opt/backups/backup-db.sh >/dev/null <<'EOF'
#!/bin/bash
set -e
cd /opt/backups || exit 1
TS=$(date +%Y%m%d_%H%M%S)
# 容器名、用户名、库名按实际改
docker exec <postgres-container> pg_dump -U <db_user> -d <db_name> \
  | gzip > postgres/<db_name>_$TS.sql.gz
find postgres -name '<db_name>_*_*.sql.gz' -mtime +14 -delete
EOF
sudo chmod +x /opt/backups/backup-db.sh

# 每天 3:17 跑(deploy 用户)
sudo tee /etc/cron.d/<app>-backup >/dev/null <<EOF
17 3 * * * deploy /opt/backups/backup-db.sh >> /opt/backups/backup.log 2>&1
EOF
```

验证:`sudo systemctl reload cron`(或 crond),手动跑一次 `/opt/backups/backup-db.sh` 看是否生成 `.sql.gz`。

## 恢复备份

```bash
# 解压 + 导回
gunzip -c /opt/backups/postgres/<db_name>_YYYYMMDD_HHMMSS.sql.gz \
  | docker exec -i <postgres-container> psql -U <db_user> -d <db_name>
```
> 恢复前先 `docker compose down backend`(避免恢复时后端在写),恢复完再起。

## certbot 自动续期

`certbot.timer` 安装时默认启用,确认即可:
```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run   # 模拟续期,验证流程通
```
证书有效期 90 天,timer 每 12 小时检查,过期前 30 天自动续。

## 容器自愈

docker-compose.yml 里所有服务加 `restart: unless-stopped`:
- 服务器重启 → 容器自动起(Docker 开机自启:`systemctl enable docker`)
- 容器崩溃 → 自动重启
- 手动 `docker compose stop` 不会自启(`unless-stopped` 尊重手动停止)

## 监控(可选,轻量)

定期检查脚本,异常发通知(可接企业微信/钉钉 webhook):
```bash
# 每分钟检查容器 + HTTPS,挂了就告警
*/1 * * * * deploy /opt/backups/healthcheck.sh
```
healthcheck.sh 示例:`docker compose ps | grep -q Exit` → 告警;`curl -fsS -m 5 https://<domain>/` 失败 → 告警。

## 交付清单模板

部署完填这张表给用户:

| 项 | 值 |
| --- | --- |
| 生产域名 | https://<domain> |
| 管理员账号 | <admin_user>(密码单独私密告知,不入文档) |
| 服务器 IP | <ip> |
| SSH 用户 | deploy(密钥) / ubuntu(sudo) |
| 证书有效期 | Let's Encrypt 90 天,自动续 |
| 备份策略 | 每天 3:17,/opt/backups/postgres/,保留 14 天 |
| CI Secrets | SSH_HOST / SSH_USER / SSH_KEY / GIT_PAT |
| CI 触发 | push 到 main 自动部署 |
| .env 位置 | /opt/<app>/.env(仅服务器,不入 git) |
