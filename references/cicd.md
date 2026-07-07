# GitHub Actions CI/CD(服务器本地构建)

## 架构决策

**不要在 runner 构镜像再跨境传**(腾讯云→服务器 ~10KB/s,226MB 镜像要几小时)。改为:workflow 用 `appleboy/ssh-action` SSH 进服务器,在**服务器本地**走代理拉代码 + `docker compose up -d --build`。

## workflow 文件 `.github/workflows/deploy.yml`

```yaml
name: Deploy to Production
on:
  push:
    branches: [main]
  workflow_dispatch:
concurrency:
  group: deploy-production
  cancel-in-progress: false
jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - name: Deploy via SSH (服务器本地构建)
        uses: appleboy/ssh-action@v1.0.3
        env:
          GIT_PAT_VALUE: ${{ secrets.GIT_PAT }}
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          port: 22
          command_timeout: 18m
          envs: GIT_PAT_VALUE
          script: |
            set -e
            cd /opt/<app>
            # 代理健康检查(隧道在线才能部署,否则 git pull 会 TLS 切断)
            curl -s --max-time 10 -x http://127.0.0.1:1080 https://api.github.com/zen -o /dev/null \
              || { echo "proxy down — 先在开发机开 ssh -R 隧道"; exit 1; }
            # PAT 直接嵌入 URL(避免 sh -c 引号转义地狱)
            git -c http.proxy=http://127.0.0.1:1080 pull --ff-only \
              https://<org>:${GIT_PAT_VALUE}@github.com/<org>/<repo>.git main
            export HTTP_PROXY=http://127.0.0.1:1080 HTTPS_PROXY=http://127.0.0.1:1080
            docker compose up -d --build
            docker image prune -f
            docker compose ps
```

> 把 `<app>` `<org>` `<repo>` 换成实际值。

## CI Secrets(仓库 Settings → Secrets and variables → Actions)

| Secret 名 | 值 |
| --- | --- |
| `SSH_HOST` | 服务器公网 IP |
| `SSH_USER` | `deploy` |
| `SSH_KEY` | deploy 用户私钥**完整内容**(含 BEGIN/END 行) |
| `GIT_PAT` | GitHub PAT |

## 为什么 PAT 嵌入 URL 而不用 credential helper

ssh-action 的 `script` 在远端 `sh -c` 执行,引号转义复杂。把 PAT 直接拼进 `https://org:TOKEN@github.com/.../repo.git` 最稳,pull 后用 `git remote set-url` 清掉(或 pull 命令本身不写 remote,TOKEN 只在当次 URL 里)。

## PAT 权限

- classic PAT:勾 `repo`(读私有仓库)+ `workflow`(触发/改 workflow)
- fine-grained PAT:**必须显式授权目标仓库**(否则 clone 403),权限给 Contents read/write + Workflows read/write
- 拿不准用 classic

## 验证 CI

push 到 main → Actions 页面看任务跑 → `conclusion=success` → 服务器 `docker compose ps` 看新容器已起。
失败先看 workflow 日志的 SSH script 输出,多数是代理隧道没开(`proxy down`)。
