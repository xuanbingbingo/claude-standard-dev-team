# app-deploy-agent

通用应用部署模板。**每次部署或更新任意应用时使用。**
依赖 `infra-bootstrap-agent` 已完成服务器初始化（含共享 `infra-mysql`）。

数据库部署策略由仓库根目录的 `deploy.yaml` 中的 `db_mode` 字段决定：
- `shared`：使用基础设施层的 `infra-mysql`（推荐）
- `dedicated`：应用自带独立 mysql 容器
- `none`：无数据库

详细 schema 见 `deploy-yaml-schema.md`。

---

## 设计思路

每个应用部署后：
- 在 `/opt/apps/{APP_NAME}/` 独立存放代码
- 加入共享 `gateway-net` Docker 网络
- **不对外暴露端口**（MySQL 除外），所有流量经中央 Nginx 网关路由
- 向 `/opt/gateway/conf.d/locations/{APP_NAME}.conf` 注册路径路由规则

```
Internet
    │
    ▼ :80
nginx-gateway (gateway-net)
    ├── /todo/      → todo-frontend:80
    ├── /todo/api/  → todo-backend:3000
    ├── /blog/      → blog-frontend:80
    └── /blog/api/  → blog-backend:3000
```

---

## 应用参数（每次填写）

| 变量 | 示例 | 说明 |
|---|---|---|
| `APP_NAME` | `todo` | 应用唯一标识，用于目录名、容器名前缀 |
| `APP_PATH` | `todo` | URL 路径前缀，每个应用唯一（访问地址为 `DOMAIN/APP_PATH/`） |
| `APP_DIR` | `/opt/apps/todo` | 服务器部署目录 |
| `LOCAL_SRC` | `/Users/libin/my-team` | 本地代码根目录 |
| `DOMAIN` | `example.com` | 所有应用共用的域名（或服务器 IP），infra-bootstrap 时已写入 main.conf |
| `FRONTEND_CONTAINER` | `todo-frontend` | 前端容器名 |
| `BACKEND_CONTAINER` | `todo-backend` | 后端容器名 |
| `DB_MODE` | `shared`/`dedicated`/`none` | 数据库部署策略，**从 deploy.yaml 读取** |
| `DB_NAME` | `todo_db` | 数据库名（shared 模式必填，从 deploy.yaml 读取） |
| `DB_VERSION` | `8.0` | 独立 MySQL 版本（dedicated 模式必填） |
| `DB_MEM_LIMIT` | `512m` | 独立 MySQL 内存上限（dedicated 模式建议，默认 512m） |
| `DB_CONTAINER` | `todo-mysql` | 仅 dedicated 模式使用，独立 MySQL 容器名 |
| `ENABLE_HTTPS` | `true/false` | 是否为共享域名申请 SSL（整个服务器只需申请一次） |

---

## 执行流程

### Phase 0：读取 deploy.yaml（决定后续分支）

```bash
# 0.1 读取仓库根 deploy.yaml
DEPLOY_YAML=$LOCAL_SRC/deploy.yaml

if [ ! -f $DEPLOY_YAML ]; then
  echo "WARN: deploy.yaml 不存在，按存量兼容模式 db_mode=dedicated 处理"
  DB_MODE=dedicated
else
  DB_MODE=$(grep '^db_mode:' $DEPLOY_YAML | awk '{print $2}' | tr -d '"'"'")
  APP_NAME=$(grep '^app_name:' $DEPLOY_YAML | awk '{print $2}' | tr -d '"'"'")
  APP_PATH=$(grep '^app_path:' $DEPLOY_YAML | awk '{print $2}' | tr -d '"'"'")
  DB_NAME=$(grep '^db_name:' $DEPLOY_YAML | awk '{print $2}' | tr -d '"'"'")
  DB_VERSION=$(grep '^db_version:' $DEPLOY_YAML | awk '{print $2}' | tr -d '"'"'")
  DB_MEM_LIMIT=$(grep '^db_mem_limit:' $DEPLOY_YAML | awk '{print $2}' | tr -d '"'"'")
  : ${DB_MEM_LIMIT:=512m}
fi

# 0.2 校验
case "$DB_MODE" in
  shared)
    [ -z "$DB_NAME" ] && { echo "FAIL: db_mode=shared 但缺少 db_name"; exit 1; }
    ;;
  dedicated)
    [ -z "$DB_VERSION" ] && echo "WARN: db_mode=dedicated 建议显式指定 db_version"
    ;;
  none)
    ;;
  *)
    echo "FAIL: db_mode 必须是 shared/dedicated/none，当前 $DB_MODE"; exit 1
    ;;
esac

echo "✓ 部署模式：db_mode=$DB_MODE"
```

---

### Phase 0.5：前缀 404 预检（首次部署必做，更新部署建议做）

> **目标**：在推送任何代码到生产前，确认前端 API 路径已正确使用环境变量前缀，杜绝子路径部署导致的 404。

```bash
# 0.5.1 检查是否存在统一 HTTP 客户端，且 baseURL 使用环境变量
grep -r "VITE_API_BASE" $LOCAL_SRC/frontend/src/ \
  || echo "❌ FAIL: 未找到 VITE_API_BASE，前端 API 路径可能硬编码"

# 0.5.2 扫描硬编码的 /api 绝对路径（最危险的问题）
# 排除 .env 文件和注释，找真正的代码调用
grep -rn "['\"]\s*/api/" $LOCAL_SRC/frontend/src/ \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.vue" \
  && echo "❌ FAIL: 发现硬编码 /api 路径，见上方文件列表" \
  || echo "✅ PASS: 未发现硬编码 /api 路径"

# 0.5.3 检查 .env 文件是否存在并包含 VITE_API_BASE
[ -f $LOCAL_SRC/frontend/.env ] \
  && grep "VITE_API_BASE" $LOCAL_SRC/frontend/.env \
  || echo "❌ FAIL: frontend/.env 缺少 VITE_API_BASE 配置"

[ -f $LOCAL_SRC/frontend/.env.production ] \
  && grep "VITE_API_BASE=/$APP_PATH" $LOCAL_SRC/frontend/.env.production \
  || echo "❌ FAIL: frontend/.env.production 缺少 VITE_API_BASE=/$APP_PATH"

# 0.5.4 检查 vite.config.ts/js 中 base 是否硬编码
if [ -f $LOCAL_SRC/frontend/vite.config.ts ] || [ -f $LOCAL_SRC/frontend/vite.config.js ]; then
  grep -n "base:" $LOCAL_SRC/frontend/vite.config.* \
    | grep -v "VITE_BASE_URL\|process.env\|import.meta.env" \
    | grep -v "^.*//.*base:" \
    && echo "❌ WARN: vite.config 中 base 可能硬编码，请检查上方输出" \
    || echo "✅ PASS: vite.config base 未发现硬编码"
fi

# 0.5.5 检查 package.json 中 homepage 是否硬编码（CRA 项目）
if [ -f $LOCAL_SRC/frontend/package.json ]; then
  grep '"homepage"' $LOCAL_SRC/frontend/package.json \
    && echo "⚠️ WARN: 检测到 homepage 字段，确认其值是否与 APP_PATH 一致" \
    || true
fi
```

**预检结果处理：**

| 结果 | 处理方式 |
|------|---------|
| 全部 PASS | 继续 Phase 1 |
| 发现硬编码 `/api` 路径 | **立即停止**，修复所有硬编码后重新预检，禁止带此问题部署 |
| 缺少 `.env.production` 或 `VITE_API_BASE` 配置 | **立即停止**，创建或补充配置后重新预检 |
| vite.config base 硬编码 | **立即停止**，改为读取环境变量后重新预检 |

**自动修复模板（发现问题时，agent 直接修复后重新预检）：**

```bash
# 创建 frontend/.env（如不存在）
cat > $LOCAL_SRC/frontend/.env << 'EOF'
VITE_API_BASE=
VITE_BASE_URL=/
EOF

# 创建 frontend/.env.production（如不存在或缺少配置）
cat > $LOCAL_SRC/frontend/.env.production << EOF
VITE_API_BASE=/$APP_PATH
VITE_BASE_URL=/$APP_PATH/
EOF
```

> ⚠️ **如果发现硬编码 `/api` 路径，必须人工定位并修改为 `\${import.meta.env.VITE_API_BASE}/api/...`**，
> agent 不得自动批量替换（可能破坏 ws://、图片路径等其他用途的路径），需上报给开发者修复。

---

### Phase 1：部署前检查

```bash
# 1.1 SSH 连通性
ssh ... "echo connected"

# 1.2 确认基础设施就绪（infra-bootstrap 已完成）
ssh ... "docker network inspect gateway-net > /dev/null 2>&1 || echo 'MISSING: run infra-bootstrap-agent first'"
ssh ... "docker ps | grep nginx-gateway || echo 'MISSING: nginx-gateway not running'"

# 1.3 判断部署类型：首次 or 更新
ssh ... "[ -d $APP_DIR ] && echo 'UPDATE' || echo 'FIRST_DEPLOY'"
# → 影响后续步骤的执行方式

# 1.4 检查应用 docker-compose.yml 是否已接入 gateway-net
# （本地检查，部署前修正）
grep -A5 "networks:" $LOCAL_SRC/docker-compose.yml | grep "gateway-net" \
  || echo "WARN: docker-compose.yml 未接入 gateway-net，需要修正"

# 1.5 按 db_mode 分支校验
case "$DB_MODE" in
  shared)
    # shared 模式禁止仓库 compose 包含 mysql service
    grep -E '^\s+(mysql|mariadb):' $LOCAL_SRC/docker-compose.yml \
      && { echo "FAIL: db_mode=shared 但 docker-compose.yml 包含 mysql service，需删除"; exit 1; } \
      || echo "✓ shared 模式：compose 无 mysql service"
    # 校验 infra-mysql 在线
    ssh ... "docker ps --format '{{.Names}}' | grep -q '^infra-mysql$'" \
      || { echo "FAIL: infra-mysql 未运行，请先确认 infra-bootstrap 完成"; exit 1; }
    ;;
  dedicated)
    grep -A10 "mysql" $LOCAL_SRC/docker-compose.yml | grep "volumes:" \
      && grep "mysql_data" $LOCAL_SRC/docker-compose.yml \
      || echo "WARN: MySQL 容器未挂载具名 volume，需要修正（防止数据丢失）"
    grep -A5 -E '^\s+(mysql|mariadb):' $LOCAL_SRC/docker-compose.yml | grep -q 'mem_limit' \
      || echo "WARN: dedicated MySQL 未设 mem_limit，强烈建议添加（默认 512m）防止 OOM"
    grep -A8 -E '^\s+(mysql|mariadb):' $LOCAL_SRC/docker-compose.yml | grep -E '"\s*[0-9]+:3306"' \
      && echo "WARN: dedicated MySQL 暴露宿主机端口可能与其他应用冲突，建议改为 SSH 隧道调试"
    ;;
  none)
    grep -E '^\s+(mysql|mariadb):' $LOCAL_SRC/docker-compose.yml \
      && echo "WARN: db_mode=none 但 docker-compose.yml 包含 mysql service，请确认"
    ;;
esac
```

---

### Phase 2：修正 docker-compose.yml（首次部署前本地执行）

确保应用的 `docker-compose.yml` 满足多应用共存要求：

**必须修改的点：**

```yaml
# ✗ 不应该暴露前端/后端端口到宿主机（会冲突）
ports:
  - "8080:80"     # ← 删除

# ✓ 改为只接入网关网络，不暴露端口
networks:
  - gateway-net

# ✗ 不应该自建 bridge 网络
networks:
  app-net:
    driver: bridge

# ✓ 改为使用外部 gateway-net
networks:
  gateway-net:
    external: true
    name: gateway-net
```

**数据库容器例外** — MySQL 可以保留宿主机端口映射（用于本地调试），但建议用非标准端口：
```yaml
ports:
  - "13306:3306"  # 只供调试，不对公网开放
```

**MySQL 必须挂载 Volume（强制）** — 防止容器删除或重建后数据丢失：
```yaml
# ✗ 不挂载 volume，容器删除后数据永久丢失
services:
  mysql:
    image: mysql:8

# ✓ 必须挂载具名 volume
services:
  mysql:
    image: mysql:8
    volumes:
      - {APP_NAME}_mysql_data:/var/lib/mysql

volumes:
  {APP_NAME}_mysql_data:
    driver: local
```

> 使用具名 volume（而非匿名 volume 或 bind mount）的原因：`docker-compose down` 默认不会删除具名 volume，数据安全；且路径与应用名绑定，易于识别和备份。

---

### Phase 3：传输代码

```bash
# 3.1 首次部署：创建目录
ssh ... "mkdir -p $APP_DIR"

# 3.2 同步代码（首次全量，更新增量）
rsync -avz --progress \
  --exclude='node_modules' \
  --exclude='.git' \
  --exclude='frontend/dist' \
  --exclude='backend/node_modules' \
  --exclude='*.log' \
  --exclude='.env*' \
  -e "ssh -i $SSH_KEY -p $SSH_PORT" \
  $LOCAL_SRC/ \
  $SSH_USER@$SERVER_IP:$APP_DIR/

# 3.3 校验关键文件
ssh ... "ls $APP_DIR/docker-compose.yml"
```

---

### Phase 4：生产环境配置

```bash
# 4.1 检查是否已有 .env 文件（更新时不覆盖已有配置）
ssh ... "[ -f $APP_DIR/.env ] && echo 'ENV_EXISTS'"

# 4.2 首次部署时生成生产配置（仅 ENV_EXISTS=false 时执行）
# 按 db_mode 分支生成
case "$DB_MODE" in
  shared)
    # 4.2a shared：在 infra-mysql 里建库 + 建账号 + 授权
    ssh ... "
      INFRA_ROOT_PWD=\$(grep MYSQL_ROOT_PASSWORD /opt/infra-mysql/.env | cut -d= -f2)
      DB_PASS=\$(openssl rand -base64 20 | tr -d '=+/' | head -c 20)
      JWT_SECRET=\$(openssl rand -base64 40 | tr -d '=+/' | head -c 40)
      docker exec infra-mysql mysql -uroot -p\"\$INFRA_ROOT_PWD\" <<-SQL
        CREATE DATABASE IF NOT EXISTS \`$DB_NAME\` CHARACTER SET utf8mb4;
        CREATE USER IF NOT EXISTS '$APP_NAME'@'%' IDENTIFIED BY '\$DB_PASS';
        ALTER USER '$APP_NAME'@'%' IDENTIFIED BY '\$DB_PASS';
        GRANT ALL PRIVILEGES ON \`$DB_NAME\`.* TO '$APP_NAME'@'%';
        FLUSH PRIVILEGES;
SQL
      cat > $APP_DIR/.env << EOF
NODE_ENV=production
JWT_SECRET=\$JWT_SECRET
DB_HOST=infra-mysql
DB_PORT=3306
DB_USER=$APP_NAME
DB_PASSWORD=\$DB_PASS
DB_NAME=$DB_NAME
EOF
      chmod 600 $APP_DIR/.env
    "
    ;;
  dedicated)
    # 4.2b dedicated：保留原逻辑，密码由 agent 生成，DB_HOST 取 compose 内 mysql service 名
    ssh ... "
      DB_PASS=\$(openssl rand -base64 20 | tr -d '=+/' | head -c 20)
      JWT_SECRET=\$(openssl rand -base64 40 | tr -d '=+/' | head -c 40)
      cat > $APP_DIR/.env << EOF
NODE_ENV=production
JWT_SECRET=\$JWT_SECRET
DB_PASSWORD=\$DB_PASS
MYSQL_PASSWORD=\$DB_PASS
MYSQL_ROOT_PASSWORD=\$(openssl rand -base64 20 | tr -d '=+/' | head -c 20)
EOF
      chmod 600 $APP_DIR/.env
    "
    ;;
  none)
    ssh ... "
      JWT_SECRET=\$(openssl rand -base64 40 | tr -d '=+/' | head -c 40)
      cat > $APP_DIR/.env << EOF
NODE_ENV=production
JWT_SECRET=\$JWT_SECRET
EOF
      chmod 600 $APP_DIR/.env
    "
    ;;
esac

# 4.3 检查 Nginx 代理配置（容器间用服务名，不用 localhost）
ssh ... "grep 'proxy_pass' $APP_DIR/frontend/nginx.conf"
# 期望：proxy_pass http://{BACKEND_CONTAINER}:{BACKEND_INNER_PORT}
```

---

### Phase 5：构建 & 启动

```bash
cd $APP_DIR

# 5.1 更新场景：优雅停止旧容器（保留数据卷）
# ssh ... "cd $APP_DIR && docker-compose stop frontend backend"

# 5.2 构建新镜像
ssh ... "cd $APP_DIR && docker-compose build --no-cache 2>&1 | tail -20"

# 5.3 启动服务
ssh ... "cd $APP_DIR && docker-compose up -d"

# 5.4 等待数据库就绪
case "$DB_MODE" in
  shared)
    # infra-mysql 已由 infra-bootstrap 起好，仅快速 ping 一次
    ssh ... "docker exec infra-mysql mysqladmin ping -h localhost --silent" \
      && echo "✓ infra-mysql alive" \
      || { echo "FAIL: infra-mysql 异常，停止部署"; exit 1; }
    ;;
  dedicated)
    ssh ... "
    for i in \$(seq 1 24); do
      STATUS=\$(docker inspect --format='{{.State.Health.Status}}' $DB_CONTAINER 2>/dev/null)
      [ \"\$STATUS\" = 'healthy' ] && echo 'DB ready' && break
      echo \"Waiting DB... (\$i/24)\"; sleep 5
    done
    "
    ;;
  none)
    echo "✓ db_mode=none，跳过数据库等待"
    ;;
esac

# 5.5 确认所有容器运行
ssh ... "docker-compose -f $APP_DIR/docker-compose.yml ps"
```

---

### Phase 6：注册到 Nginx 网关

向 `conf.d/locations/` 写入该应用的路径路由，不修改 `main.conf`：

```bash
ssh ... "cat > /opt/gateway/conf.d/locations/$APP_NAME.conf << 'EOF'
# [$APP_NAME] 前端
location /$APP_PATH/ {
    rewrite ^/$APP_PATH/(.*)$ /\$1 break;
    proxy_pass http://$FRONTEND_CONTAINER:80;
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
}

# [$APP_NAME] 后端 API（rewrite 去掉路径前缀，后端无需感知）
location /$APP_PATH/api/ {
    rewrite ^/$APP_PATH(/api/.*)$ \$1 break;
    proxy_pass http://$BACKEND_CONTAINER:3000;
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_http_version 1.1;
    proxy_set_header Connection '';
}
EOF"
```

> `rewrite` 的作用：nginx 收到 `/todo/api/v1/todos` → 转发给后端时变成 `/api/v1/todos`，
> 后端代码无需知道自己被部署在 `/todo/` 路径下。

**HTTPS（整个服务器只申请一次，由 infra-bootstrap 或首次部署时完成）：**
```bash
# 申请证书（仅在 ENABLE_HTTPS=true 且证书不存在时执行）
ssh ... "[ -d /etc/letsencrypt/live/$DOMAIN ] || \
  certbot certonly --standalone --non-interactive \
  --agree-tos --email $CERTBOT_EMAIL -d $DOMAIN \
  --pre-hook 'docker stop nginx-gateway' \
  --post-hook 'docker start nginx-gateway'"

# 更新 main.conf 支持 HTTPS（首次 HTTPS 应用部署时修改一次）
ssh ... "cat > /opt/gateway/conf.d/main.conf << 'EOF'
server {
    listen 80 default_server;
    server_name $DOMAIN;
    return 301 https://\$host\$request_uri;
}
server {
    listen 443 ssl default_server;
    server_name $DOMAIN;
    ssl_certificate /etc/letsencrypt/live/$DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    include /etc/nginx/conf.d/locations/*.conf;
}
EOF"
```

**重载网关配置（零停机）：**
```bash
ssh ... "docker exec nginx-gateway nginx -t && docker exec nginx-gateway nginx -s reload"
```

---

### Phase 7：验收

```bash
# 7.1 应用健康检查（通过路径前缀访问）
ssh ... "curl -sf http://localhost/$APP_PATH/api/health | head -c 200"
# 期望：{"status":"ok"}

# 7.2 前端页面可访问
ssh ... "curl -sf -o /dev/null -w '%{http_code}' http://localhost/$APP_PATH/"
# 期望：200

# 7.3 查看已注册的路由规则
ssh ... "ls /opt/gateway/conf.d/locations/"

# 7.4 输出摘要
echo "============================="
echo "✓ $APP_NAME 部署完成"
echo "  地址：http://$DOMAIN/$APP_PATH/"
echo "  目录：$APP_DIR"
echo "  容器："
ssh ... "docker ps | grep $APP_NAME"
echo "============================="
```

---

### Phase 8：生成 GitHub Actions CI/CD 工作流（首次部署后执行）

> **目标**：首次部署完成后，在项目仓库中生成 `.github/workflows/deploy.yml`，
> 后续日常代码更新直接在 GitHub 页面手动触发，无需再本地 rsync。

**触发条件**：首次部署成功（Phase 7 验收通过）且 `.github/workflows/deploy.yml` 不存在时执行。

```bash
# 8.1 检查是否已存在工作流文件
[ -f $LOCAL_SRC/.github/workflows/deploy.yml ] && echo "SKIP: workflow already exists" && exit 0

# 8.2 创建目录
mkdir -p $LOCAL_SRC/.github/workflows

# 8.3 生成工作流文件
cat > $LOCAL_SRC/.github/workflows/deploy.yml << EOF
name: Deploy

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Sync code to server
        uses: burnett01/rsync-deployments@7.0.1
        with:
          switches: >-
            -avz --delete
            --exclude='.env*'
            --exclude='node_modules'
            --exclude='**/node_modules'
            --exclude='.git'
            --exclude='frontend/dist'
            --exclude='*.log'
          path: ./
          remote_path: $APP_DIR/
          remote_host: \${{ secrets.SERVER_HOST }}
          remote_user: \${{ secrets.SERVER_USER }}
          remote_key: \${{ secrets.SERVER_SSH_KEY }}

      - name: Rebuild and restart
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: \${{ secrets.SERVER_HOST }}
          username: \${{ secrets.SERVER_USER }}
          key: \${{ secrets.SERVER_SSH_KEY }}
          script: |
            set -e
            cd $APP_DIR
            docker compose up --build -d
            docker compose ps
            docker exec nginx-gateway nginx -s reload
EOF
```

**8.4 输出 GitHub Secrets 配置清单：**

```
===========================
请在 GitHub 仓库 → Settings → Secrets and variables → Actions 中添加以下 Secrets：

  SECRET 名称        值
  ─────────────────────────────────────────
  SERVER_HOST        {SERVER_IP}
  SERVER_USER        {SSH_USER}（通常为 root）
  SERVER_SSH_KEY     ~/.ssh/id_rsa 的完整内容（cat ~/.ssh/id_rsa）

配置完成后，访问仓库 → Actions → Deploy → Run workflow 即可触发部署。
===========================
```

> **注意**：
> - 工作流不传输 `.env*` 文件，服务器上的 `.env` 由首次部署（Phase 4）生成后永久保留
> - `--delete` 参数会删除服务器上本地没有的文件，`.env` 因被 exclude 不受影响
> - 若项目无前端（纯后端），删除 `--exclude='frontend/dist'` 行

---

## 错误处理策略

| 场景 | 处理方式 |
|---|---|
| **前端存在硬编码 `/api` 路径（Phase 0.5）** | **立即停止，上报具体文件和行号，等待开发者修复后重新部署，不得绕过** |
| **缺少 `.env.production` 或 `VITE_API_BASE` 未设置（Phase 0.5）** | 自动创建 `.env.production` 并写入正确配置，重新预检后继续 |
| **vite.config base 硬编码（Phase 0.5）** | 立即停止，上报问题，等待开发者修复后重新部署 |
| `gateway-net` 不存在 | 终止，提示先运行 `infra-bootstrap-agent` |
| `infra-mysql` 不存在但 `db_mode=shared` | 终止，提示先运行 `infra-bootstrap-agent` 完成 Phase 4.5 |
| 仓库无 `deploy.yaml` | 按 `db_mode=dedicated` 兼容存量项目继续，但提醒补齐 |
| `db_mode=shared` 但仓库 compose 含 mysql service | 终止，提示删除后重新部署（数据迁移走 deploy-yaml-schema.md 流程） |
| `db_mode=dedicated` 且 mysql 暴露 `3306:3306` | 警告，仅推荐改为 SSH 隧道调试，不强制阻塞 |
| `db_mode=dedicated` 且 mysql 未设 `mem_limit` | 警告，建议补齐防 OOM，不强制阻塞 |
| `docker-compose.yml` 未接入 `gateway-net` | 本地自动修正后重新传输 |
| MySQL 容器未挂载具名 volume（dedicated） | 本地自动添加 volume 配置后重新传输，**不得跳过，数据丢失不可恢复** |
| 容器启动失败 | `docker logs` 抓最后 50 行，分析后上报 |
| 数据库 120s 未 healthy | 输出诊断日志，提示检查内存/磁盘 |
| Nginx 配置测试失败 | 不执行 reload，保留旧配置，上报错误原因 |
| 证书申请失败 | 降级为 HTTP 部署，提示手动处理 DNS |
| rsync 失败 | 重试一次，仍失败则尝试 scp |

---

## 多应用端口分配约定

各应用容器有独立网络命名空间，**后端内部端口统一用 3000**，不会冲突。
唯一需要规划的是数据库宿主机调试端口（每个应用不同）：

| 应用 | 后端内部端口 | 数据库宿主机调试端口 |
|---|---|---|
| todo | 3000 | 13306 |
| blog | 3000 | 13307 |
| admin | 3000 | 13308 |
| ... | 3000 | 1330x |

---

## Agent Prompt 模板

```
你是一个经验丰富的 DevOps 工程师。
你的任务是将一个应用部署到已初始化的云服务器。

## 服务器信息
- IP: {SERVER_IP}
- SSH 用户: {SSH_USER}
- SSH 密钥: {SSH_KEY}

## 应用信息
- APP_NAME: {APP_NAME}
- APP_PATH: {APP_PATH}
- 本地代码路径: {LOCAL_SRC}
- 共享域名: {DOMAIN}
- 访问地址: http://{DOMAIN}/{APP_PATH}/
- 前端容器: {FRONTEND_CONTAINER}
- 后端容器: {BACKEND_CONTAINER}
- 包含数据库: {HAS_DATABASE}
- 启用 HTTPS: {ENABLE_HTTPS}

## 执行原则
1. Phase 1 先判断是首次部署还是更新，后续步骤据此调整
2. 每步先检查状态再操作（幂等）
3. 修改 docker-compose.yml 前先备份（cp docker-compose.yml docker-compose.yml.bak）
4. Nginx 配置变更前必须先 nginx -t 测试通过再 reload
5. 遇到错误立即停止并报告

## 执行步骤
按照 app-deploy-agent.md Phase 1 → Phase 8 顺序执行。
```
