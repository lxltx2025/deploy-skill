# deploy-skill

> 将 **NestJS + Prisma + Postgres + Vite/React** 全栈应用部署到腾讯云 Ubuntu 轻量服务器的生产级 ZCode skill —— 完成 Docker 化、Nginx 反代、HTTPS 证书、GitHub Actions CI/CD、SMTP 邮箱验证码登录的全套生产落地与排错。

## 它解决什么

标准部署步骤谁都会查,这个 skill 的价值在于**踩过的坑的标准化排错**:

- 国内服务器直连 GitHub 的 TLS 被 `GnuTLS recv error (-110)` 切断 → 反向代理隧道方案
- 跨境传镜像 ~10KB/s 不可行 → 服务器本地构建架构
- `.cloud/.site` 等 TLD 触发 SNI 阻断,站点打不开 → `.com` 域名 + 诊断方法
- seed 检测"已存在数据"跳过,账号不同步 → 手动 UPDATE 数据库
- SMTP 变量没加载,验证码只进日志不发邮件 → `--env-file .env` 重启
- 私钥指纹不匹配 / PAT 权限不足 / 端口冲突 … 40+ 条现象→根因→解法

## Skill 结构

```
deploy-skill/
├── SKILL.md                    主流程:11 阶段 + 关键约束 + 校验标准
└── references/                 渐进式披露,按需读取
    ├── env-template.md         .env 模板 + 强随机密钥生成 + 端口绑定
    ├── nginx-certbot.md        Nginx 反代配置 + HTTPS 证书申请
    ├── cicd.md                 GitHub Actions workflow + CI Secrets
    ├── troubleshooting.md      40+ 条故障现象→根因→解法对照表
    └── ops.md                  数据库备份 / 证书续期 / 容器自愈 / 交付清单
```

## 安装

把本仓库克隆到 ZCode 的 skills 目录之一即可被发现:

```bash
# 用户级(跨项目可用,推荐)
git clone https://github.com/lxltx2025/deploy-skill.git ~/.agents/skills/deploy-skill

# 或项目级(仅当前仓库)
git clone https://github.com/lxltx2025/deploy-skill.git .zcode/skills/deploy-skill
```

## 使用

安装后,在 ZCode 里说这些话会自动触发:

- "帮我把 github.com/org/app 部署到腾讯云 82.x.x.x,要 HTTPS 和邮箱登录"
- "网站打不开,域名访问 connection reset"
- "本地能登录服务器不行"
- "GitHub Actions 部署失败了"

也可手动强制加载:`/skill deploy-skill <你的需求>`

## 前置条件

- 腾讯云 Ubuntu 22.04+ 轻量服务器 + sudo 权限
- `.com` 域名(已解析到服务器)
- GitHub 私有仓库 + PAT
- 开发机本机 Clash(提供出口代理给服务器,解决国内访问 GitHub 被切)
- SMTP 邮箱授权码(验证码登录)

完整参数清单见 `SKILL.md` 的「前置条件」表。
