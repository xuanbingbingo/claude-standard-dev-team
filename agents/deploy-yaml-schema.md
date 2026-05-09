# deploy.yaml Schema

每个应用在仓库根目录维护一个 `deploy.yaml`，`app-deploy-agent` 据此决定部署策略。
首次部署时由 agent 引导生成（若缺失），后续由开发者维护。

---

## 字段定义

| 字段 | 必填 | 类型 | 取值 / 默认 | 说明 |
|---|---|---|---|---|
| `app_name` | ✓ | string | 小写英文 + 连字符 | 应用唯一标识，决定容器名前缀和服务器目录名 |
| `app_path` | ✓ | string | 小写英文 + 连字符 | URL 路径前缀，全服务器唯一（访问地址 `DOMAIN/{app_path}/`） |
| `db_mode` | ✓ | enum | `shared` / `dedicated` / `none` | 数据库部署策略，详见下方 |
| `db_name` | shared 时必填 | string | 下划线 / 小写 | 在 `infra-mysql` 内要建的库名 |
| `db_version` | dedicated 时必填 | string | 例：`8.0` / `5.7` | 独立 MySQL 镜像版本 |
| `db_mem_limit` | dedicated 时建议 | string | 默认 `512m` | 独立 MySQL 容器的 `mem_limit`，强制存在以防 OOM |
| `frontend_container` | ✓ | string | 例：`<app>-frontend` | 前端容器名 |
| `backend_container` | ✓ | string | 例：`<app>-backend` | 后端容器名 |
| `backend_port` | | int | 默认 `3000` | 后端在容器内监听的端口 |
| `enable_https` | | bool | 默认 `false` | 是否申请 SSL（首次启用时由 agent 触发 certbot） |

---

## db_mode 三种模式行为对照

### `shared`（推荐用于普通业务）

`app-deploy-agent` 会自动：

1. 在 `infra-mysql` 中执行（幂等）：
   ```sql
   CREATE DATABASE IF NOT EXISTS {db_name} CHARACTER SET utf8mb4;
   CREATE USER IF NOT EXISTS '{app_name}'@'%' IDENTIFIED BY '{随机密码}';
   GRANT ALL PRIVILEGES ON {db_name}.* TO '{app_name}'@'%';
   FLUSH PRIVILEGES;
   ```
   注：用户名超过 32 字符时取 `{app_name}` 截断后缀，详细规则在 agent 实现里处理。

2. 把账号密码写入服务器 `/opt/apps/{app_name}/.env`（已存在则不覆盖）：
   ```env
   DB_HOST=infra-mysql
   DB_PORT=3306
   DB_USER={app_name}
   DB_PASSWORD={随机密码}
   DB_NAME={db_name}
   ```

3. 校验仓库 `docker-compose.yml`：
   - **不得**包含任何 `mysql` / `mariadb` service（包含则报错并停止）
   - backend service **必须**加入 `gateway-net`

4. 备份策略：由 `infra-mysql` 侧统一 `mysqldump --all-databases`，应用侧无需配置。

### `dedicated`（敏感数据、特殊版本、高写入压力）

`app-deploy-agent` 会自动：

1. 不动 `infra-mysql`。
2. 校验仓库 `docker-compose.yml`：
   - **必须**包含一个 mysql service
   - mysql service **不暴露**宿主机端口（`ports:` 不允许出现 `3306`）
   - mysql service **必须**挂载具名 volume `{app_name}_mysql_data`
   - mysql service **必须**设 `mem_limit`（取 `db_mem_limit`，默认 512m）
3. 生成 .env 时按应用 compose 内 mysql service 名作为 `DB_HOST`，密码继续用 agent 生成。
4. 备份策略：由应用方在自己的部署目录维护备份脚本（agent 提供模板）。

### `none`（纯前端 / 纯静态 / 无状态服务）

`app-deploy-agent` 会自动：

1. 不创建任何 mysql 资源。
2. .env 不写入数据库相关字段。
3. 跳过 Phase 5.4「等待数据库 healthy」。

---

## 示例

**shared**：
```yaml
app_name: shequ
app_path: shequ
db_mode: shared
db_name: shequ_db
frontend_container: shequ-frontend
backend_container: shequ-backend
enable_https: true
```

**dedicated**：
```yaml
app_name: legacy-erp
app_path: erp
db_mode: dedicated
db_version: '5.7'
db_mem_limit: 384m
frontend_container: erp-frontend
backend_container: erp-backend
```

**none**：
```yaml
app_name: docs
app_path: docs
db_mode: none
frontend_container: docs-frontend
backend_container: docs-backend
```

---

## 兼容存量项目

仓库未带 `deploy.yaml` 时，`app-deploy-agent` 默认按 `db_mode: dedicated` 处理（即"维持现状"），不主动迁移。
要把存量项目迁到 `shared`，需开发者：

1. 在仓库根加 `deploy.yaml`，`db_mode: shared` + `db_name`。
2. 导出旧 mysql 数据：`docker exec {old-mysql} mysqldump -uroot -p... {db} > dump.sql`
3. 导入到 infra-mysql：`docker exec -i infra-mysql mysql -uroot -p... {db} < dump.sql`
4. 在仓库 `docker-compose.yml` 删掉 mysql service，backend 加入 `gateway-net`。
5. 更新服务器 `/opt/apps/{app_name}/.env` 的 DB_HOST、密码。
6. 重启 backend；验证后下线老 mysql 容器和 volume。

迁移期间建议短停服窗口，避免双写导致数据不一致。
