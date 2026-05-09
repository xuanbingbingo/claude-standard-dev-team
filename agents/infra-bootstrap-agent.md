# infra-bootstrap-agent

服务器基础设施初始化。**每台服务器只运行一次。**
完成后，所有应用通过 `app-deploy-agent` 部署，共享本文建立的基础设施。

---

## 前置条件（人工确认）

- [ ] 云服务器已创建，可 SSH 登录
- [ ] 安全组已开放：22（SSH）、80（HTTP）、443（HTTPS）
- [ ] 服务器内存 ≥ 1GB，磁盘 ≥ 20GB
- [ ] 已有域名并完成 DNS 解析到服务器 IP（若需 HTTPS）

---

## 服务器信息（启动前填写）

| 变量 | 值 | 说明 |
|---|---|---|
| `SERVER_IP` | | 公网 IP |
| `SSH_USER` | `root` | SSH 用户名 |
| `SSH_KEY` | `~/.ssh/id_rsa` | 本地密钥路径 |
| `SSH_PORT` | `22` | SSH 端口 |
| `ENABLE_HTTPS` | `true/false` | 是否申请 SSL 证书 |
| `CERTBOT_EMAIL` | | Let's Encrypt 邮箱（HTTPS 时必填） |

---

## 执行流程

### Phase 1：环境探测

```bash
# 1.1 测试 SSH 连通性
ssh -i $SSH_KEY -p $SSH_PORT -o ConnectTimeout=10 \
  -o StrictHostKeyChecking=no $SSH_USER@$SERVER_IP "echo connected"

# 1.2 检测操作系统
ssh ... "cat /etc/os-release | grep -E '^ID=|^VERSION_ID='"
# → ubuntu / debian / centos / rhel / almalinux

# 1.3 检查是否已初始化（幂等保障）
ssh ... "docker --version 2>/dev/null && echo 'docker_exists'"
ssh ... "[ -f /opt/gateway/docker-compose.yml ] && echo 'gateway_exists'"
# 已初始化 → 跳过对应步骤，不重复安装
```

---

### Phase 2：安装 Docker

**Ubuntu / Debian：**
```bash
apt-get update -y
apt-get install -y ca-certificates curl gnupg lsb-release
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  > /etc/apt/sources.list.d/docker.list
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**CentOS / AlmaLinux / RHEL：**
```bash
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**安装后验证：**
```bash
systemctl start docker && systemctl enable docker
docker run --rm hello-world
# 安装 docker-compose 独立版（兼容旧命令）
curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \
  -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

---

### Phase 3：建立 Docker 网络

```bash
# 所有应用容器和网关共享此网络
docker network create --driver bridge gateway-net 2>/dev/null || echo "network exists"
```

> **约定**：每个应用的 `docker-compose.yml` 必须将前端/后端容器加入 `gateway-net`，
> 网关通过容器名路由，不暴露宿主机端口（MySQL 除外）。

---

### Phase 4：部署中央 Nginx 网关

```bash
mkdir -p /opt/gateway/conf.d/locations /opt/gateway/certs /opt/gateway/logs
```

**/opt/gateway/docker-compose.yml：**
```yaml
version: '3.8'
services:
  nginx-gateway:
    image: nginx:alpine
    container_name: nginx-gateway
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - ./logs:/var/log/nginx
    networks:
      - gateway-net

networks:
  gateway-net:
    external: true
    name: gateway-net
```

**/opt/gateway/conf.d/main.conf：**
```nginx
# 未匹配任何路径时默认拒绝
server {
    listen 80 default_server;
    return 444;
}

# 主服务器：所有应用共享，各应用通过 locations/ 注册路径
server {
    listen 80;
    server_name {DOMAIN};               # 启动前替换为实际域名或 IP

    # 各应用部署时自动写入此目录
    include /etc/nginx/conf.d/locations/*.conf;
}
```

> **约定**：每个应用部署时只向 `conf.d/locations/` 添加自己的 location 文件，不修改 `main.conf`。

```bash
cd /opt/gateway && docker-compose up -d
# 验证（此时 locations/ 为空，主 server 无路径，返回 404）
curl -sf -o /dev/null -w "%{http_code}" http://localhost:80
# 期望：404（说明网关正常运行，等待应用注册路径）
```

---

### Phase 4.5：部署共享 MySQL（基础设施层）

> **目标**：在基础设施层提供一个共享 MySQL 实例（容器名 `infra-mysql`），供采用 `shared` 数据库模式的应用使用。
> 应用通过仓库根目录的 `deploy.yaml` 选择 `shared` / `dedicated` / `none`，详见 `deploy-yaml-schema.md`。

```bash
mkdir -p /opt/infra-mysql/conf.d
```

**/opt/infra-mysql/.env**（首次创建时随机生成 root 密码，永不进 git）：
```bash
[ -f /opt/infra-mysql/.env ] || cat > /opt/infra-mysql/.env << EOF
MYSQL_ROOT_PASSWORD=$(openssl rand -base64 24 | tr -d '=+/' | head -c 24)
EOF
chmod 600 /opt/infra-mysql/.env
```

**/opt/infra-mysql/conf.d/custom.cnf**（限制内存占用，避免全局 OOM）：
```ini
[mysqld]
innodb_buffer_pool_size = 128M
innodb_log_buffer_size  = 16M
max_connections         = 100
performance_schema      = OFF
table_open_cache        = 256
tmp_table_size          = 16M
max_heap_table_size     = 16M
```

**/opt/infra-mysql/docker-compose.yml**：
```yaml
version: '3.8'
services:
  infra-mysql:
    image: mysql:8
    container_name: infra-mysql
    restart: unless-stopped
    env_file: .env
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - infra_mysql_data:/var/lib/mysql
      - ./conf.d:/etc/mysql/conf.d:ro
    mem_limit: 512m
    networks:
      - gateway-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  infra_mysql_data:
    driver: local

networks:
  gateway-net:
    external: true
    name: gateway-net
```

```bash
cd /opt/infra-mysql && docker-compose up -d
# 等 healthy
for i in $(seq 1 18); do
  STATUS=$(docker inspect --format='{{.State.Health.Status}}' infra-mysql 2>/dev/null)
  [ "$STATUS" = 'healthy' ] && echo '✓ infra-mysql ready' && break
  echo "Waiting infra-mysql... ($i/18)"; sleep 5
done
```

> **约定**：
> - `infra-mysql` **不暴露宿主机端口**，仅在 `gateway-net` 内部可达，本地调试通过 SSH 隧道。
> - 应用 `db_mode: shared` 时，由 `app-deploy-agent` 在 `infra-mysql` 里建库 + 建账号 + 仅授权本库（最小权限）。
> - 应用 `db_mode: dedicated` 时，应用自带独立 mysql 容器，**不得暴露宿主机端口**（避免冲突），必须设 `mem_limit ≤ 512m`。
> - 升级 / 重启 `infra-mysql` 前需评估对所有 shared 业务的影响，建议低峰期操作。

---

### Phase 5：安装 Certbot（HTTPS，可选）

仅当 `ENABLE_HTTPS=true` 时执行：

**Ubuntu：**
```bash
apt-get install -y certbot
```

**CentOS：**
```bash
yum install -y epel-release && yum install -y certbot
```

> 证书申请在各应用部署时按域名单独申请，不在此阶段执行。

---

### Phase 6：基础安全加固

```bash
# 6.1 配置 UFW 防火墙（Ubuntu）
which ufw && {
  ufw allow 22/tcp
  ufw allow 80/tcp
  ufw allow 443/tcp
  ufw --force enable
  ufw status
}

# 6.2 firewalld（CentOS）
which firewall-cmd && {
  firewall-cmd --permanent --add-service=ssh
  firewall-cmd --permanent --add-service=http
  firewall-cmd --permanent --add-service=https
  firewall-cmd --reload
}

# 6.3 设置服务器时区
timedatectl set-timezone Asia/Shanghai

# 6.4 配置 Docker 日志大小限制（防止日志撑爆磁盘）
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
systemctl restart docker
```

---

### Phase 7：验收

```bash
# 基础设施全部就绪检查
docker --version
docker-compose --version
docker network inspect gateway-net
docker ps | grep nginx-gateway
docker ps | grep infra-mysql
docker exec infra-mysql mysqladmin ping -h localhost -uroot -p"$(grep MYSQL_ROOT_PASSWORD /opt/infra-mysql/.env | cut -d= -f2)" 2>&1 | grep -q alive && echo "✓ infra-mysql alive"
systemctl is-active docker

echo "============================="
echo "✓ 基础设施初始化完成"
echo "✓ 中央 Nginx 网关运行在 80/443"
echo "✓ Docker 网络 gateway-net 就绪"
echo "✓ 共享 MySQL（infra-mysql）就绪，root 密码见 /opt/infra-mysql/.env"
echo "✓ 接下来使用 app-deploy-agent 部署各应用"
echo "============================="
```

---

## Agent Prompt 模板

```
你是一个经验丰富的 DevOps 工程师。
你的任务是初始化一台全新的云服务器，为后续部署多个应用做好基础设施准备。

## 服务器信息
- IP: {SERVER_IP}
- SSH 用户: {SSH_USER}
- SSH 密钥: {SSH_KEY}
- 系统: 待自动探测
- 是否启用 HTTPS: {ENABLE_HTTPS}

## 执行原则
1. 每步先检查是否已完成，已完成则跳过（保证幂等）
2. 遇到错误立即停止并报告，不跳过
3. 所有远程操作通过 Bash 工具执行 ssh 命令
4. 每个 Phase 完成输出一行 ✓ 状态确认

## 执行步骤
按照 infra-bootstrap-agent.md Phase 1 → Phase 7 顺序执行。

## 禁止行为
- 不得修改 sshd_config
- 不得 kill -9 任何进程
- 不得删除服务器上已有的应用目录
```
