# SteeringFlow Website 发布 Runbook

> 适用范围：仅 `steeringflow.com`（含 `www.steeringflow.com`）静态官网。
> **不适用**于 `app.steeringflow.com`（Cloud 应用，由同一 Nginx 容器的另一组 vhost + 后端容器提供服务）。
>
> 本文档基于 2026-10-14 的只读审计编写。所有命令均来自实际核验的环境；
> 未核实项已明确标注 `UNKNOWN` / `BLOCKED`，执行前须由维护者确认。

---

## 1. 架构与托管方式（已验证）

| 组件 | 事实 | 证据 |
|---|---|---|
| 域名 | `steeringflow.com`、`www.steeringflow.com` 指向同一台服务器 | nginx vhost `server_name` |
| 服务器 | 腾讯云 Ubuntu VM，`43.131.247.132`，SSH 22 端口，用户 `ubuntu` | 本次 SSH 审计 |
| Web 服务 | Docker 容器 `repo-nginx-1`（`nginx:alpine`），发布 80/443 | `docker ps`、`ss -tlnp` |
| Compose 栈 | 项目名 `repo`，工作目录 `/opt/steeringflow/repo-iss07`，`-f` 组合：`docker-compose.cloud.yml` + `docker-compose.yml` + `docker-compose.production.yml` + `docker-compose.website.yml` | `docker compose ls` |
| 站点根目录 | 容器内 `/usr/share/nginx/html/website`（**只读挂载**宿主机 `/opt/steeringflow/website-dist`） | `docker inspect` Mounts、`docker-compose.website.yml` |
| vhost 配置 | 由 `_deploy/nginx/website-vhost.conf.template` 渲染生成，安装到 `/opt/steeringflow/repo-iss07/_deploy/nginx/runtime/rendered/website-vhost.conf`，该目录挂载为容器 `/etc/nginx/conf.d` | 容器内 `nginx -T`、Mounts |
| HTTP 行为 | `:80` 全站 301 跳转 `https://$host$request_uri`（**保留 host**：www 跳 www，apex 跳 apex） | vhost 配置 + curl 实测 |
| HTTPS 行为 | apex 与 www 均返回 200，内容相同 | curl 实测（2026-10-14） |
| 安全策略 | 拒绝点文件（`.git` 等，`.well-known` 除外）；`/tls/` 返回 404；webroot 为 git-archive 产物、**不含 `.git`** | vhost 配置注释与规则 |
| TLS 证书 | 宿主机 `/opt/steeringflow/tls/website/{fullchain,privkey}.pem`，只读挂载到容器 `/etc/nginx/website-tls`；由 `certbot-renew.timer`（每日 00:31）自动续期 | Mounts、`systemctl list-timers` |
| ACME | webroot 模式，挑战目录 `/opt/steeringflow/website-dist/.well-known/acme-challenge` | vhost、`renewal-steeringflow.com.conf` |
| GitHub 仓库 | `https://github.com/SteeringFlow/website`（public，默认分支 `main`） | `git remote -v` + GitHub 页面 |
| CI/CD | **无**。GitHub Pages 未启用，无 Actions workflow，仓库内无部署脚本 | GitHub 页面 + 仓库文件列表 |

## 2. Git 分支、远程与发布源

- 本地仓库即官网仓库（`origin` = `https://github.com/SteeringFlow/website.git`），分支 `main`。
- 发布源契约（服务器端 `deploy-website.sh` 强制）：
  - 必须显式指定 `--repo <url>`（临时 clone）或 `--src <dir>`（显式本地仓库路径）；
  - **无默认源，未指定即 fail closed**；
  - 强烈建议同时用 `--ref <sha>` 固定提交。
- 服务器上**不存在**官网的持久 git 工作副本（source contract 明确禁止生产依赖 working copy）。

## 3. 发布前检查（只读）

1. 本地官网仓库：
   ```bash
   git fetch origin
   git status                 # 工作树应干净
   git log --oneline origin/main -3
   git branch -vv             # 确认 ahead/behind，无分叉
   ```
2. 确认目标提交已推送到 `origin/main`：
   ```bash
   git merge-base --is-ancestor <sha> origin/main && echo PUSHED || echo NOT_PUSHED
   ```
3. 服务器预检（不安装、不 reload）：
   ```bash
   REPO_DIR=/opt/steeringflow/repo-iss07 \
     /opt/steeringflow/repo-iss07/_deploy/deploy-website.sh \
     --repo https://github.com/SteeringFlow/website \
     --ref <sha> --check
   ```
   通过标准：clone 成功、`nginx -t: PASS`、无 `__WEBSITE_*` 残留。
4. 记录当前线上版本（回滚锚点）：
   ```bash
   cat /opt/steeringflow/website-dist.source 2>/dev/null   # 可能不存在，见 §8
   git -C /opt/steeringflow/website-dist hash-object index.html styles.css
   ```
   将输出与本地 `git rev-parse <sha>:index.html` 比对，确认线上对应 commit。

## 4. 正式发布（按序执行）

> 所有服务器命令的执行位置：SSH 登录 `ubuntu@43.131.247.132`。
> `REPO_DIR=/opt/steeringflow/repo-iss07` **必须显式设置**：脚本默认值为
> `/opt/steeringflow/repo`（旧副本，其 rendered 目录并非线上挂载目录）。

```bash
# 0) 本地：推送提交（如未推送）
git push origin main

# 1) 服务器：发布前手动备份（脚本"先清空再解包"，中断会留下不完整产物）
cp -a /opt/steeringflow/website-dist \
      /opt/steeringflow/website-dist.bak.$(date +%Y%m%d-%H%M%S)

# 2) 服务器：正式发布（git archive 解包到 website-dist + 渲染 vhost + nginx -t + reload）
REPO_DIR=/opt/steeringflow/repo-iss07 \
  /opt/steeringflow/repo-iss07/_deploy/deploy-website.sh \
  --repo https://github.com/SteeringFlow/website \
  --ref <sha>

# 3) 服务器：核对 provenance
cat /opt/steeringflow/website-dist.source
```

脚本行为（已验证源码）：
1. `git clone --repo URL` 到临时目录 → 解析 `--ref` 为 commit；
2. `find <DST> -mindepth 1 -delete` 清空 `/opt/steeringflow/website-dist`，再 `git archive <commit> | tar -x` 解包（产物无 `.git`、无未跟踪文件）；
3. 重建 `.well-known/acme-challenge`；
4. 写 provenance 到 `/opt/steeringflow/website-dist.source`（source/ref/commit/deployed_at）；
5. 渲染 vhost（替换 `__WEBSITE_TLS_CERT__` / `__WEBSITE_TLS_KEY__`，有残留即失败）；
6. 隔离 `nginx -t` 门禁（`docker run nginx:alpine`，只读挂载 TLS 目录）；
7. 安装 rendered vhost，`docker exec repo-nginx-1 nginx -s reload`。

幂等：可重复执行；`--check` 只验证不安装；`--skip-nginx` 不 reload。

## 5. 发布后 HTTPS 验证清单

```bash
# 重定向与状态码
curl -sS -o /dev/null -w '%{http_code} -> %{redirect_url}\n' http://steeringflow.com/
#   期望: 301 -> https://steeringflow.com/
curl -sS -o /dev/null -w '%{http_code} -> %{redirect_url}\n' http://www.steeringflow.com/
#   期望: 301 -> https://www.steeringflow.com/
curl -sS -o /dev/null -w 'apex: %{http_code}\n' https://steeringflow.com/     # 期望 200
curl -sS -o /dev/null -w 'www:  %{http_code}\n' https://www.steeringflow.com/  # 期望 200

# 内容抽查（按本次发布预期调整关键字）
curl -sS https://steeringflow.com/ | grep -c 'footer-contact'      # 期望 >= 1
curl -sS https://steeringflow.com/ | grep -o 'hello@steeringflow.com' | head -1
curl -sS https://steeringflow.com/ | grep -o 'mailto:hello@steeringflow.com' | head -1

# provenance
cat /opt/steeringflow/website-dist.source
```

浏览器人工检查：
- [ ] 首页正常渲染，无 404/混合内容
- [ ] EN / 中文切换均显示正确文案
- [ ] 桌面端与移动端（窄屏）布局无溢出
- [ ] 顶部导航与 GitHub 链接正常
- [ ] `www.steeringflow.com` 与 apex 行为一致
- [ ] 顺带确认 `app.steeringflow.com` 未受 reload 影响（同一 nginx 容器，reload 为优雅重启，理论上无感）

## 6. 常见失败与诊断

| 现象 | 原因与处理 |
|---|---|
| `git clone failed` | 服务器出网受限。先重试；若持续，由维护者决定是否改用 `--src <显式路径>`（仍需满足 source contract） |
| `website TLS cert missing` | 证书未就绪。按脚本提示先执行 `/opt/steeringflow/scripts/certbot-deploy.sh`（需维护者确认其当前适用性） |
| `nginx -t failed` | 查看输出；多为模板被改坏或 TLS 文件损坏 |
| `rendered vhost has __WEBSITE_* residue` | 模板文件被破坏，fail closed，从仓库副本恢复模板 |
| `nginx reload failed` | `docker ps` 检查 `repo-nginx-1` 状态 |
| 网站 404 / 样式丢失 | `website-dist` 内容不完整（发布中断）。执行 §7 回滚 |
| 页面为旧版本 | 确认 `website-dist.source` 的 commit 与预期一致；浏览器强刷排除缓存 |

## 7. 回滚

**回滚前提**：已知良好的 commit SHA（本 Runbook 记录：当前线上为 `d64b830`，
内容 = "fix(website): point Cloud Early Access CTA to app instead of mailto"，
已于 2026-10-14 通过 blob hash 逐文件验证）。

**回滚步骤**（重新发布旧版本，而非恢复文件）：
```bash
REPO_DIR=/opt/steeringflow/repo-iss07 \
  /opt/steeringflow/repo-iss07/_deploy/deploy-website.sh \
  --repo https://github.com/SteeringFlow/website \
  --ref <known-good-sha>
```

**备用回滚**（脚本不可用时）：恢复 §4 步骤 1 的手动备份目录并 reload：
```bash
# 仅作应急手段；执行前须确认备份目录内容完整
cp -a /opt/steeringflow/website-dist.bak.<timestamp>/. /opt/steeringflow/website-dist/
docker exec repo-nginx-1 nginx -s reload
```

**回滚能力现状（不过度声称）**：
- 无自动化网站备份。`/opt/steeringflow/backups/` 仅含数据库每日 dump（`acp_daily_*`），**不含网站文件**。
- `/opt/steeringflow/website.retired.20260921.tar.gz` 为旧快照（含 `.git` 的完整检出），其 `index.html` 与当前线上相同，可作溯源参考，**不建议**直接作为回滚源。
- 因此：**每次发布前必须执行 §4 的手动备份**，否则无可靠回滚点。

## 8. 凭据管理与安全注意事项

- 本文档**不记录**任何私钥、密码、Token、htpasswd。
- SSH 私钥由维护者本地保管（本机路径不在文档中记录）；服务器 `ubuntu` 用户有免密 sudo 与 docker 组权限。
- 官网仓库为 **public**：严禁向该仓库提交任何敏感信息。
- webroot 内不得出现 `.git`、TLS 私钥（vhost 已有点文件 deny 与 `/tls/` 404 兜底，但仍不应依赖）。
- 修改 vhost 只走"模板 → deploy-website.sh 渲染"路径，不要手工编辑 rendered 文件（会被下次发布覆盖）。

## 9. 已知限制与待维护者确认事项

1. `UNKNOWN`：`/opt/steeringflow/website-dist.source` 不存在。现网产物与 `d64b830`
   逐文件 blob 一致（内容已证实），但**产生该产物的具体动作无法最终确认**
   （可能为旧版脚本或人工上传）。不影响后续按本 Runbook 发布。
2. `UNKNOWN`：服务器 `git clone https://github.com/SteeringFlow/website` 的出网连通性
   未实测（`--check` 会实际 clone，发布前预检即可验证）。
3. `注意`：脚本默认 `REPO_DIR=/opt/steeringflow/repo`（旧副本 Sep-21）。
   必须显式传 `REPO_DIR=/opt/steeringflow/repo-iss07`（线上挂载的 rendered 目录所属副本）。
4. `注意`：nginx reload 为容器级操作，`app.steeringflow.com` 与官网同容器，
   reload 有极短窗口影响两者（优雅 reload，预期无感；已记录依赖关系）。
5. `BLOCKED`：截至本 Runbook 编写时，本地提交 `2a1c9f0`
   （feat(website): add early access contact information）**尚未推送** `origin/main`，
   也**未发布**。首次正式发布时须先完成 `git push origin main`。
6. 建议后续：将"发布前手动备份 `website-dist`"固化为脚本步骤或定时任务
   （当前无自动网站备份，属运维缺口，需维护者决策）。

---

*审计与文档生成：2026-10-14。只读审计，未对生产做任何变更。*
