---
name: deploy-skill
description: Deploy a NestJS+Prisma+Postgres+Vite/React fullstack app to a Tencent Cloud Ubuntu Lighthouse server end-to-end — Docker, Nginx reverse proxy, HTTPS via certbot, GitHub Actions CI/CD, and SMTP email-code login. Use whenever the user mentions deploying to Tencent Cloud (腾讯云/轻量服务器), GitHub Actions auto-deploy, HTTPS/证书, SNI 阻断, docker compose, Nginx 反向代理, SMTP 验证码登录, or says "部署到服务器/上线/生产". Also use when the user reports a deployed site is unreachable, login fails, or CI/CD breaks — the skill's troubleshooting table maps symptoms to root causes.
---

# 全栈应用部署到腾讯云 Ubuntu(生产级)

## 这个 skill 做什么

把一套 **NestJS(后端 3000) + Vite/React(前端,容器内 nginx 80) + Postgres** 的全栈应用,从 GitHub 私有仓库部署到腾讯云 Ubuntu 轻量服务器,完成 Docker 化、Nginx 反代、HTTPS 证书、GitHub Actions 自动部署、SMTP 邮箱验证码登录,达到国内用户可访问、生产可用。

它最大的价值不在"标准步骤"(那些谁都会查),而在**踩过的坑的标准化排错**:国内服务器直连 GitHub 的 TLS 被切断、跨境传镜像不可行、seed 跳过导致账号不同步、SMTP 变量没加载、SNI 阻断让站点打不开——这些都内置了解法。

## 适用边界

**适用**:腾讯云 Ubuntu 22.04+ 轻量服务器、GitHub 私有仓库 + Docker Compose、需要国内访问 + HTTPS + 邮箱验证码。

**不适用(需调整)**:非 Ubuntu / 非腾讯云(命令需改)、无 sudo 权限、境外服务器(无 SNI 阻断,流程可简化)、无 SSH 密钥/密码访问。

## 关键约束(执行前必读)

这些是反复踩坑总结的硬约束,违反必出问题:

1. **全程走反向代理隧道**。腾讯云服务器直连 GitHub 的 `git clone` 会被 `GnuTLS recv error (-110)` 切断(API 短请求能通,git 长数据流不行)。必须在开发机本机开 `ssh -R 0.0.0.0:1080:127.0.0.1:7890`(Clash 7890)反代给服务器,Docker daemon、git pull、npm 全部走 `127.0.0.1:1080`。
2. **跨境传镜像不可行**(~10KB/s)。CI/CD 架构改为**服务器本地构建** + 走代理拉代码/依赖,不要在 runner 构镜像再传。
3. **域名必须用 `.com`**。`.cloud/.site` 等境外 TLD 会触发 SNI 阻断,国内用户打不开。用 IP 直连能开、用域名 `curl -v` 看到 `Connection reset` 就是 SNI 问题。
4. **容器端口绑 `127.0.0.1`**。Postgres/后端/前端一律 `127.0.0.1:xxxx`,绝不裸暴露公网,只通过宿主机 Nginx 透传。
5. **宿主机 Nginx 只透传前端容器 8080**。前端容器内置的 nginx 已处理 `/api` 和 `/socket.io`,宿主机别重复配,否则路由打架。
6. **凭据全部占位符化 / 随机生成**。DB 密码、JWT 密钥、admin 密码用 `openssl rand` 生成,绝不硬编码。.env 仅存服务器本地,不入 git。
7. **CI/CD 依赖三件套常开**:开发机 + Clash + `ssh -R` 隧道。隧道断了 CI 就失败,部署前先 `curl -x http://127.0.0.1:1080 https://api.github.com/zen` 验证。

## 前置条件(缺了再问用户要)

| 参数 | 必填 | 含义 | 示例 |
| --- | :--: | --- | --- |
| `server_ip` | ✅ | 服务器公网 IP | `82.156.149.89` |
| `ssh_user` | ✅ | SSH 用户(初始 ubuntu / CI 用 deploy) | `ubuntu` |
| `deploy_key` | ✅ | deploy 用户 SSH 私钥(CI 用,完整内容) | `-----BEGIN OPENSSH...` |
| `repo_url` | ✅ | GitHub 私有仓库 HTTPS URL | `https://github.com/org/repo.git` |
| `git_pat` | ✅ | GitHub PAT(clone/pull 私有仓库) | `ghp_xxx` / `github_pat_xxx` |
| `domain` | ✅ | 国内可解析的 `.com` 域名 | `example.com` |
| `db_password` | ✅ | 生产 DB 密码(`openssl rand` 生成) | — |
| `jwt_secret` | ✅ | JWT 密钥(`openssl rand -hex 32`) | — |
| `smtp_*` | ✅ | SMTP 配置(验证码登录) | host/port/user/pass/from |
| `letsencrypt_email` | ✅ | certbot 注册邮箱 | `you@example.com` |
| `ssh_password` | 选填 | 初始密码登录(装密钥用) | — |
| `proxy_port` | 选填 | 反向隧道落点端口 | `1080`(默认) |

**PAT 注意**:fine-grained PAT 必须显式授权目标仓库,否则 clone 403;拿不准就用 classic PAT(`repo` 作用域)。CI 还需 `workflow` 作用域触发。

## 执行流程(11 阶段)

按顺序执行,每阶段有"通过判据",没过就查下方排错表。

### 阶段 0:环境探测(先诊断,不假设)

先确认 DNS、SSH、SNI 三件事,再动手:
```bash
dig +short @223.5.5.5 <domain> A          # 国内 DNS 解析
dig +short @8.8.8.8 <domain> A            # 国外 DNS 解析(应一致)
timeout 8 bash -c 'cat </dev/null >/dev/tcp/<server_ip>/22' && echo "SSH 可达"
curl -v https://<domain>/                 # Recv failure: Connection reset → SNI 阻断
curl -v https://<server_ip>/              # 通 → 印证是 SNI 非服务器问题
```

### 阶段 1:建立 SSH 访问(密钥优先)

1. 本地生成 deploy 密钥对:`ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -N "" -C "deploy@ci"`
2. 首次密码登录装公钥到 `/home/deploy/.ssh/authorized_keys`(权限 700/600)
3. **指纹双端比对**(避免私钥传输损坏):本地 `ssh-keygen -lf ~/.ssh/deploy_key` 与服务器端必须一致
4. 被拒时查 sshd DEBUG:`LogLevel DEBUG3` → `systemctl reload ssh` → 观察后恢复 INFO

### 阶段 2:建立代理隧道(核心,见关键约束 1)

开发机本机执行(保持终端常开):
```bash
ssh -R 0.0.0.0:1080:127.0.0.1:7890 \
    -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes \
    deploy@<server_ip>
```
服务器端验证:`curl -x http://127.0.0.1:1080 https://api.github.com/zen` 返回 200。

### 阶段 3:服务器环境初始化

- 系统 + Docker(官方脚本 `get.docker.com`)+ `docker-compose-plugin`
- 建 `deploy` 用户并加 docker 组,建 `/opt/<app>` 并 chown
- Docker daemon 走代理(`/etc/systemd/system/docker.service.d/http-proxy.conf` 指 `127.0.0.1:1080`)→ daemon-reload + restart

### 阶段 4:克隆私有仓库(走代理 + PAT)

**约束**:deploy 无权直接写 /opt(root 所有),先 clone 到 home 再 mv:
```bash
sudo -u deploy git -c http.proxy=http://127.0.0.1:1080 clone \
  https://<org>:<git_pat>@github.com/<org>/<repo>.git /home/deploy/repo-clone
sudo cp -a /home/deploy/repo-clone/. /opt/<app>/ && sudo chown -R deploy:deploy /opt/<app>
cd /opt/<app> && sudo -u deploy git remote set-url origin https://github.com/<org>/<repo>.git  # 清除 URL 中的 PAT
```

### 阶段 5:配置 .env(强随机密钥)

DB/JWT 密钥全部 `openssl rand` 生成,**不要硬编码**。完整 .env 模板见 `references/env-template.md`。生成后 `chmod 600`,属主 deploy。

### 阶段 6:docker-compose 端口绑定

所有容器端口加 `127.0.0.1:` 前缀(postgres 5432 / backend 3000 / frontend 8080),不裸暴露公网。

### 阶段 7:构建并启动(走代理拉依赖)

```bash
cd /opt/<app>
sudo -E env HTTP_PROXY=http://127.0.0.1:1080 HTTPS_PROXY=http://127.0.0.1:1080 \
  docker compose up -d --build
```

### 阶段 8:Nginx 反向代理 + HTTPS

宿主机 Nginx **只透传到 `127.0.0.1:8080`**(前端容器),别重复配 `/api`。完整 nginx 配置见 `references/nginx-certbot.md`,关键点:upstream keepalive 32、proxy_http_version 1.1、Upgrade/Connection 头(支持 websocket)、`proxy_read_timeout 86400s`、`client_max_body_size 50m`。certbot 非交互申请 + `--redirect` 强制 HTTPS。

### 阶段 9:种子账号与邮箱验证码登录

**关键陷阱**:服务器 DB 首次启动 seed 创建账号后,再改 seed 代码 push,日志会显示"已存在数据,跳过"——新账号不同步,需手动 UPDATE 数据库:
1. 容器内 `bcryptjs.hashSync` 生成新密码 hash
2. `psql` UPDATE user 表:username/password/email/mustChangePassword/loginFailCount/lockedUntil
3. **重启 backend 必须 `--env-file .env`**,否则 SMTP 变量为空,验证码只进日志不发邮件
4. 触发登录测 SMTP:`POST /api/auth/login` → 查邮箱收到验证码 + 日志无 error

### 阶段 10:GitHub Actions CI/CD(服务器本地构建)

架构决策见关键约束 2。`.github/workflows/deploy.yml` 用 `appleboy/ssh-action` SSH 进服务器,先代理健康检查(隧道在线才部署),PAT 直接嵌入 pull URL(避免 sh -c 引号转义地狱),`docker compose up -d --build`。完整 workflow + CI Secrets 清单见 `references/cicd.md`。

### 阶段 11:运维兜底

- 数据库每日备份(保留 14 天):`pg_dump | gzip` cron 脚本(模板见 `references/ops.md`)
- certbot 自动续期:`certbot.timer` 默认启用,确认即可
- 容器 `restart: unless-stopped` 自愈

## 校验标准(部署完成判据)

| 校验项 | 期望 | 验证命令 |
| --- | --- | --- |
| 容器健康 | 3 容器 Up | `docker compose ps` |
| 后端 | HTTP 200 | `curl http://127.0.0.1:3000/api` |
| 前端 nginx | HTTP 200 | `curl http://127.0.0.1:8080/` |
| HTTPS 域名 | 200 + TLS 有效 | `curl https://<domain>/` |
| HTTP→HTTPS | 301 跳转 | `curl -I http://<domain>/` |
| SMTP 邮件 | 验证码真实发送 | 触发登录查邮箱 + 日志无 error |
| CI/CD | push 后 success | GitHub Actions 页面 |

## 出问题了?先查排错表

故障现象 → 根因 → 解法的完整对照表在 `references/troubleshooting.md`。**遇到任何报错/异常先读那个文件**,它覆盖了 GnuTLS 切断、git 凭证、seed 跳过、SMTP 空、私钥指纹、SNI 阻断、端口冲突、TS 报错等所有已踩过的坑。如果排错表里没有,再新排查并把根因+解法补进去(让 skill 越用越强)。

## 交付规范

部署完成后交付:
1. **可访问站点** `https://<domain>`(HTTP 自动跳 HTTPS)
2. **管理员凭证**(强随机或指定密码,**不要在文档里明文留真实密码**)
3. **架构说明**:端口绑 127.0.0.1、宿主 Nginx 透传 8080、前端容器内 nginx 处理 /api/socket.io
4. **CI/CD 工作流** `.github/workflows/deploy.yml` 已入库
5. **运维兜底**:DB 每日备份、certbot 续期、容器自愈
6. **交付清单表**:账号/域名/证书有效期/备份策略/Secrets 名称

**必须告知用户的注意事项**:
- ⚠️ 生产 .env 仅存服务器本地,换服务器需重新生成
- ⚠️ CI/CD 依赖开发机 + Clash + ssh -R 隧道三者常开,缺一 CI 失败
- ⚠️ 代码热修(如临时注释的模块)需用户后续补全或正式删除
- ⚠️ PAT 权限过大时建议收窄为 fine-grained 最小权限
