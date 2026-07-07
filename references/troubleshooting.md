# 故障排查表(出问题先读这个)

遇到任何报错/异常,先在这张表里找现象。它覆盖了所有已踩过的坑。**如果遇到表里没有的新坑,排查清楚后把"现象→根因→解法"补进这张表**,让 skill 越用越强。

## 网络 / 代理 / Git

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| `GnuTLS recv error (-110)` | 服务器直连 GitHub TLS 被切 | 全程走 `127.0.0.1:1080` 代理(ssh -R 隧道) |
| `fatal: could not read Username` | git pull 无凭证 | PAT 嵌入 URL:`https://org:TOKEN@github.com/...` |
| git clone 报 403 | fine-grained PAT 未授权该仓库 | 用 classic PAT,或 fine-grained 显式授权目标仓库 |
| 服务器 `curl github` 通但 `git clone` 断 | API 短请求通、git 长数据流被切 | 同上,走代理 |
| CI 报 `proxy down` | 开发机 ssh -R 隧道没开 / Clash 没开 | 开发机重连 `ssh -R 0.0.0.0:1080:127.0.0.1:7890 deploy@<ip>` |
| 镜像传输极慢(~10KB/s) | 跨境传镜像不可行 | 改服务器本地构建,不要 runner 构完再传 |

## SSH / 密钥

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| `Permission denied (publickey)` | 私钥指纹与服务器 authorized_keys 不匹配 | 双端 `ssh-keygen -lf` 比对,不一致则重装公钥 |
| 私钥指纹不匹配 | 私钥文本传输/粘贴损坏(换行/空格) | 服务器本地 `ssh-keygen` 重生成一对,私钥取回 |
| 装了公钥还被拒 | 权限错(.ssh 非 700 / authorized_keys 非 600 / 属主错) | `chmod 700 ~/.ssh && chmod 600 authorized_keys && chown -R deploy:deploy ~/.ssh` |
| 定位不了哪步失败 | 默认日志太简 | `LogLevel DEBUG3` → `systemctl reload ssh` → 看完恢复 INFO |

## Docker / 容器

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| `docker load: unrecognized image format` | 用了 `docker save | gzip` 压缩 tar | 改用 `docker save -o x.tar` 未压缩(本地构建则无此问题) |
| `scp: *.tar.gz No such file` | scp 通配符在 SSH 引号内不展开 | 显式列文件名,不用通配符 |
| `Could not allocate port` 端口冲突 | 前次容器未释放 | `docker compose down` 后重启 |
| 后端镜像缺失 | 上次构建被中断 | 重新 `docker compose build backend` |
| 拉镜像超时 | Docker daemon 没走代理 | 配 `http-proxy.conf` 指 127.0.0.1:1080 + daemon-reload |
| 镜像 226MB 太大 | node_modules 含 devDependencies | runner 用 `npm ci --production`(优化,非阻塞) |

## 数据库 / Seed / 登录

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| 改了 seed 代码 push,账号没更新 | seed 检测"已存在数据"跳过(日志 `⏭️ 已存在数据`) | 手动 UPDATE 数据库,别依赖 seed |
| 本机能登录服务器不行 | 代码库密码与服务器 DB 密码不一致(本地/服务器 DB 独立,git 不同步数据) | DB UPDATE password(bcrypt 重 hash)+ 检查 seed 跳过 |
| 登录 500 / 找不到用户 | user 表是空的或 email 是假邮箱 | `SELECT username,email FROM "user"` 核对,UPDATE 真实 email |

## SMTP / 验证码

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| SMTP 未配置 | compose 没加载 .env | `docker compose --env-file .env up -d backend` |
| 验证码只进日志不发邮件 | SMTP_HOST/USER/PASS 为空 | `docker exec <backend> env \| grep SMTP_` 确认非空 + 检查 .env |
| 邮箱收不到验证码 | 用户 email 是 `@example.com` 假邮箱 | UPDATE user 表填真实邮箱 |
| SMTP 报 535 认证失败 | 用了登录密码而非授权码 | 邮箱后台开 SMTP 取授权码,填 SMTP_PASS |
| 邮件进垃圾箱 | 发件人名/域名信誉 | SPF/DKIM 记录 + 让用户加白名单 |

## HTTPS / 域名 / SNI

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| 用户报"网站打不开" | 国内运营商对 `.cloud/.site` 等 TLD 的 SNI 阻断 | 改用 `.com` 域名根治;应急用 Clash 走代理或 IP 直连 |
| 域名打不开但 IP 能开 | 同上,SNI 审查 | `curl -v https://<domain>/` 看 `Connection reset` → 判定 SNI |
| certbot 申请失败 | 80 没放行 / DNS 没解析 | 安全组放 80 + `dig` 确认解析到本机 |
| 证书过期 | certbot.timer 没启用 | `systemctl status certbot.timer` + 手动 `certbot renew --dry-run` |

## 构建 / 代码

| 现象 | 根因 | 解法 |
| --- | --- | --- |
| LogsModule TS2307 报错 | 仓库引用了未创建的模块 | 注释 app.module.ts 引用,加 `[DEPLOY-HOTFIX]` 标记,交付时提醒用户补全 |
| build 后白屏 | 前端 VITE_API_BASE_URL 没配 | .env 配 `VITE_API_BASE_URL=/api` + 重新 build |

## 排查通用思路

1. **先分层定位**:是网络(代理/SNI)、容器(Docker/端口)、应用(DB/SMTP/代码)、还是 CI(隧道/凭证)?
2. **看日志**:容器 `docker compose logs <svc> --tail 100`、nginx `tail /var/log/nginx/error.log`、sshd DEBUG
3. **最小复现**:`curl 127.0.0.1:3000/api`(绕过 nginx 测后端)、`curl 127.0.0.1:8080`(测前端容器)
4. **对比本地 vs 服务器**:DB 数据独立,git 不同步数据;配置差异是高发区
