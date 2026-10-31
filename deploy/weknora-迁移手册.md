# WeKnora 单机 Docker 部署迁移手册

> 适用场景：当前在一台服务器上用 Docker（docker compose）单机部署了 WeKnora，已有业务数据（用户、空间、知识库、**已上传的文档**），现需将整套环境（含数据与已上传文档）迁移到另一台服务器。
>
> 版本说明：本手册基于仓库 `docker-compose.yml`（paradedb/pg17 + WeKnora v0.x）。如后续 compose 结构有变动，请以实际 `docker-compose.yml` 的 `services` / `volumes` 段落为准。

---

## 1. 迁移原理（先读这一段）

WeKnora 单机 Docker 的全部**持久化数据**都在两类地方：

1. **Docker 命名卷（named volumes）**：数据库、向量、上传文件等。
2. **宿主机配置文件（bind mount）**：`.env` 与 `config/config.yaml` 等。

只要把「卷」和「配置文件」原样搬到新服务器，再按相同方式启动，迁移即完成。**没有任何数据需要手动导出到 SQL 之外的格式**（除非你选择更稳妥的 `pg_dump` 方案，见 §4.3）。

### 1.1 各服务与数据卷盘点

默认（不带任何 `--profile`）启动的服务只有 5 个，且**有持久化数据的卷只有 `postgres-data` 和 `data-files`**：

| 服务 | 镜像 | 持久化卷 | 内容 |
|---|---|---|---|
| `frontend` | weknora-ui | 无 | Nginx 静态资源，无状态 |
| `app` | weknora-app | `data-files`、`docreader-tmp`、`./config/config.yaml`(bind) | 后端，挂载上传文件目录与配置 |
| `docreader` | weknora-docreader | `docreader-tmp` | 文档解析，临时产物 |
| `postgres` | paradedb/paradedb:v0.22.6-pg17 | `postgres-data` | **核心数据库**：用户、空间、知识库元数据、文档切片，**以及默认向量 embeddings** |
| `redis` | redis:7.0-alpine | **无**（容器可写层，`--appendonly yes` 但无卷挂载） | 缓存 / Asynq 任务队列，**临时、无需迁移** |

`docker-compose.yml` 中声明的全部命名卷：

```text
postgres-data        # 核心数据库（必迁）
data-files           # 上传的原始文档文件（必迁）
docreader-tmp        # 解析临时产物（建议迁，可重建）
minio_data           # 仅 --profile minio / full
neo4j-data           # 仅 --profile neo4j / full（知识图谱）
qdrant_data          # 仅 --profile qdrant / full（向量库）
milvus_data          # 仅 --profile milvus
weaviate_data        # 仅 --profile weaviate
doris_fe_meta / doris_fe_log / doris_be_storage / doris_be_log  # 仅 --profile doris
langfuse_clickhouse_data / langfuse_clickhouse_logs / langfuse_minio_data  # 仅 --profile langfuse / full
searxng_config       # 仅 --profile searxng / full
```

> **原则：你旧服务器实际启动过哪些服务（profile），对应的卷就全部要备份；没用到的 profile 对应的卷不存在，跳过即可。**

### 1.2 两个最关键的事实

- **默认 `RETRIEVE_DRIVER=postgres`**：向量 embeddings 直接存在 PostgreSQL 里。因此只要迁 `postgres-data`，知识库检索能力就完整带走了。如果你在 `.env` 里把 `RETRIEVE_DRIVER` 改成了 `qdrant` / `milvus` / `weaviate` 等，那对应的向量卷（`qdrant_data` 等）**也必须备份**，否则文档元数据在、向量丢了、检索为空。
- **`STORAGE_TYPE` 默认 `local`**：上传文件落在 `data-files` 卷（`LOCAL_STORAGE_BASE_DIR=/data/files`）。如果你改成了 `minio` / `s3` / `oss` 等对象存储，原始文件在对象存储里，本地只需迁数据库元数据 + 对应对象存储桶，无需迁 `data-files`。

### 1.3 必须原样带走的配置文件

| 文件 | 说明 | 是否必迁 |
|---|---|---|
| `<项目根>/.env` | 所有环境变量（含密钥） | **必迁，且密钥不得重新生成** |
| `<项目根>/config/config.yaml` | 主配置文件（bind mount 进 app） | **必迁** |
| `<项目根>/config/builtin_models.yaml` | 若你创建过此文件 | 用了就迁 |
| `<项目根>/config/models.json` | 若你创建过此文件 | 用了就迁 |

> ⚠️ **`.env` 里的 `SYSTEM_AES_KEY` 是库内敏感字段（模型 API Key、向量库凭证、Web 搜索 Key 等）的 AES-256 主密钥。** 新服务器若使用不同的 `SYSTEM_AES_KEY`，旧库里这些加密字段将无法解密（UI 显示为空白，需重新填写）。因此 `.env` 必须**整文件照拷**，不要在新服务器上重新生成密钥。

其它同样建议原样保留的密钥变量：`JWT_SECRET`、`SYSTEM_SIGNING_KEY`、`DB_PASSWORD`、`REDIS_PASSWORD`。

---

## 2. 迁移前准备与检查清单

在旧服务器上完成以下确认（建议先记录到一张纸上）：

- [ ] **确认启动命令 / profile**：回忆或查看旧服务器是用 `docker compose up -d` 还是带了 `--profile full` / `--profile qdrant` 等。不确定的话执行下面命令看实际运行的容器：
      ```bash
      docker ps --format '{{.Names}}\t{{.Image}}'
      ```
      容器名前缀 `WeKnora-` 后的名字即服务名；`postgres`/`redis`/`qdrant` 等是否在列，决定要迁哪些卷。

- [ ] **确认数据卷实际名称**（compose 默认用「项目目录名」作前缀）：
      ```bash
      docker volume ls | grep -i weknora
      ```
      典型输出：`WeKnora_postgres-data`、`WeKnora_data-files`、`WeKnora_docreader-tmp` …（前缀 `WeKnora` 来自目录名）。**记下这个前缀**，新服务器恢复时要用同样的卷名。

- [ ] **确认 `.env` 关键变量**：`WEKNORA_VERSION`（镜像版本，建议新旧一致）、`RETRIEVE_DRIVER`、`STORAGE_TYPE`、`SYSTEM_AES_KEY` 是否存在且已填写。

- [ ] **确认无进行中的任务**：在 Web UI 上确认没有 `processing` 状态的文档解析任务（迁移前停机可避免丢半截任务）。

- [ ] **确认新服务器可装 Docker**：目标机器需支持 Docker Engine + Docker Compose v2（插件版 `docker compose`）。

---

## 3. 旧服务器：停机

进入项目根目录，优雅停机以保证数据落盘：

```bash
cd /path/to/WeKnora
bash scripts/start_all.sh -s
# 或等价命令：docker compose down --remove-orphans
```

> 如果你用了 profile，停机也要带相同 profile，例如 `docker compose --profile full down --remove-orphans`。
> 停机后**不要删除卷**（我们要备份它们）。

---

## 4. 旧服务器：备份

### 4.1 备份数据卷（推荐：卷快照 tar 方式）

对每个需要迁移的卷，用临时 `busybox` 容器把卷内容打成 tar：

```bash
mkdir -p /tmp/wk-backup && cd /tmp/wk-backup

# 把下面列表换成 §2 中你实际查到的卷名
VOLUMES="WeKnora_postgres-data WeKnora_data-files WeKnora_docreader-tmp"

for v in $VOLUMES; do
  docker run --rm -v "$v":/volume -v "$(pwd)":/backup busybox \
    tar czf /backup/"$v".tar.gz -C /volume .
done

ls -lh /tmp/wk-backup
```

生成形如 `WeKnora_postgres-data.tar.gz` 的备份文件。

> 说明：busybox tar 以数值 UID 保留文件属主，目标容器用相同镜像（postgres 内部 UID 999 等），还原后权限一致，无需手动 `chown`。

### 4.2 备份配置文件

```bash
cd /path/to/WeKnora
mkdir -p /tmp/wk-backup/config
cp .env /tmp/wk-backup/config/.env
cp config/config.yaml /tmp/wk-backup/config/config.yaml
# 若用过下列文件也一并拷贝
[ -f config/builtin_models.yaml ] && cp config/builtin_models.yaml /tmp/wk-backup/config/
[ -f config/models.json ] && cp config/models.json /tmp/wk-backup/config/
```

### 4.3 （备选，更稳妥）用 pg_dump 备份 PostgreSQL

直接拷数据目录在遇到镜像大版本差异时偶有风险。若想更稳妥，可改用逻辑备份（前提是 postgres 容器仍可用）：

```bash
cd /path/to/WeKnora
# 只停应用，保留 postgres 运行
docker compose stop app frontend docreader

# 先把 .env 里的变量加载进来（DB_USER / DB_NAME / DB_PASSWORD 来自 .env.example 默认值）
set -a; source .env; set +a

# 导出
docker compose exec -T postgres \
  pg_dump -U "$DB_USER" -d "$DB_NAME" -Fc > /tmp/wk-backup/weknora_db.dump

docker compose stop postgres   # 备份完再停
```

此方式在 §6.3 有对应的还原步骤。

### 4.4 打包并传输到新服务器

```bash
cd /tmp
tar czf wk-migration.tar.gz -C /tmp/wk-backup .
scp wk-migration.tar.gz user@新服务器IP:/tmp/
# 同时把项目代码也传过去（或在新服务器 git clone 同版本）
scp -r /path/to/WeKnora user@新服务器IP:/opt/WeKnora
```

> 项目代码（含 `docker-compose.yml`）新旧必须一致或兼容；建议新服务器使用**同一个 git 提交 / 同一个 `WEKNORA_VERSION`**，避免 schema 漂移。

---

## 5. 新服务器：环境准备

1. 安装 Docker Engine 与 Docker Compose v2：
   ```bash
   # 以 Ubuntu 为例（具体见 Docker 官方文档）
   curl -fsSL https://get.docker.com | sh
   docker compose version   # 确认 v2 插件可用
   ```

2. 把项目代码放到**与旧服务器同名目录**（非常重要，见 §7.1）：
   ```bash
   # 推荐：目录名保持一致，例如 /opt/WeKnora
   ls /opt/WeKnora/docker-compose.yml   # 确认存在
   ```

3. 解压迁移包：
   ```bash
   mkdir -p /tmp/wk-restore && cd /tmp/wk-restore
   tar xzf /tmp/wk-migration.tar.gz
   ls -R /tmp/wk-restore
   ```

4. 还原配置文件到项目根：
   ```bash
   cd /opt/WeKnora
   cp /tmp/wk-restore/config/.env ./.env
   cp /tmp/wk-restore/config/config.yaml ./config/config.yaml
   [ -f /tmp/wk-restore/config/builtin_models.yaml ] && cp /tmp/wk-restore/config/builtin_models.yaml ./config/
   [ -f /tmp/wk-restore/config/models.json ] && cp /tmp/wk-restore/config/models.json ./config/
   ```

---

## 6. 新服务器：恢复数据

按你在 §2 确认的实际卷名操作。以下以默认前缀 `WeKnora_` 为例。

### 6.1 卷快照方式还原（对应 §4.1）

```bash
cd /tmp/wk-restore
VOLUMES="WeKnora_postgres-data WeKnora_data-files WeKnora_docreader-tmp"

for v in $VOLUMES; do
  docker volume create "$v"                       # 先建空卷
  docker run --rm -v "$v":/volume -v "$(pwd)":/backup busybox \
    tar xzf /backup/"$v".tar.gz -C /volume        # 再灌入
done
```

> 还原时 postgres 容器必须**未运行**（我们尚未 `up`，故满足）。

### 6.2 启动服务（务必带相同 profile）

```bash
cd /opt/WeKnora
docker compose up -d                 # 默认五件套
# 若旧服务器用了 profile，这里也必须带，例如：
# docker compose --profile full up -d
```

`AUTO_MIGRATE=true`（默认）会在启动时自动执行前向数据库迁移，向上兼容新版本 schema，无需手动处理。

### 6.3 （若用 pg_dump）还原数据库

如果你在 §4.3 用的是逻辑备份，则先只起 postgres，再还原：

```bash
cd /opt/WeKnora
docker compose up -d postgres        # 仅起数据库
sleep 10                             # 等其就绪

# 先加载 .env 变量（DB_USER / DB_NAME）
set -a; source .env; set +a

docker compose exec -T postgres \
  pg_restore -U "$DB_USER" -d "$DB_NAME" --clean --if-exists /dev/stdin \
  < /tmp/wk-restore/weknora_db.dump
# 然后起全部服务
docker compose up -d
```

---

## 7. 验证与关键注意事项

### 7.1 ⚠️ 目录名 / `COMPOSE_PROJECT_NAME` 必须一致

Docker compose 默认用**项目目录的 basename** 作为卷名前缀。旧服务器目录 `WeKnora` → 卷 `WeKnora_postgres-data`；若新服务器放在 `/opt/weknora`，卷名会变成 `weknora_postgres-data`，与 §6.1 建的卷对不上，启动后会**新建空卷、数据全无**。

解决二选一：
- **推荐**：新服务器用相同目录名（如 `/opt/WeKnora`）；
- 或显式统一前缀：在项目根目录的 `.env` 里追加一行 `COMPOSE_PROJECT_NAME=WeKnora`（docker compose 会自动读取），或执行前 `export COMPOSE_PROJECT_NAME=WeKnora`。这样不论目录叫什么，卷名前缀都固定为 `WeKnora_`。

### 7.2 ⚠️ `SYSTEM_AES_KEY` 必须原样

见 §1.3。新服务器 `.env` 必须来自旧服务器拷贝，切勿重新生成，否则库内加密凭据无法解密。

### 7.3 ⚠️ 向量库驱动决定要不要额外迁卷

- `RETRIEVE_DRIVER=postgres`（默认）→ 向量在 `postgres-data`，迁它即可。
- 改为 `qdrant`/`milvus`/`weaviate` → 必须额外备份恢复对应卷（`qdrant_data` 等）。
- 知识图谱 `NEO4J_ENABLE=true` → 必须迁 `neo4j-data`。

### 7.4 ⚠️ 镜像版本

`.env` 的 `WEKNORA_VERSION` 建议与旧服务器一致。跨大版本迁移前，先确认目标版本对当前 schema 的前向迁移是否覆盖；`AUTO_MIGRATE` 只做前向，不做降级。

### 7.5 `redis` 无卷，属正常

任务队列 / 缓存会被清空。迁移前确保无进行中的文档解析任务（§2），停机后无需处理 redis。

### 7.6 验证清单

```bash
docker compose ps                       # 所有服务 Healthy/Up
curl -f http://localhost:8080/health    # app 健康检查
```

功能验证：
- [ ] 用原账号能登录（JWT 正常，说明 `JWT_SECRET` 一致）
- [ ] 空间、用户列表完整
- [ ] 之前上传的文档在知识库列表中存在，且能正常**检索/问答**（验证向量随 postgres 一并迁过来了）
- [ ] 模型配置中的 API Key 没有变成空白（验证 `SYSTEM_AES_KEY` 一致）
- [ ] 若用对象存储，验证文件可正常预览/下载

---

## 8. 常见问题排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 启动后卷是空的 / 数据丢失 | 新服务器目录名与旧不同，卷前缀不匹配 | 见 §7.1，用相同目录名或 `COMPOSE_PROJECT_NAME` |
| 能登录但模型 API Key 显示空白 | `SYSTEM_AES_KEY` 与旧不一致 | 用旧 `.env` 原样覆盖，重启 app |
| 文档在列表里但检索不到 / 答非所问 | 向量库卷未迁移（非 postgres 驱动） | 补迁 `qdrant_data` 等对应卷后重启 |
| postgres 起不来 / 数据目录报错 | 卷快照跨大版本不兼容 | 改用 §4.3 `pg_dump` 逻辑备份还原 |
| 上传的文件 404 | `data-files` 卷未迁移或路径错 | 确认 `data-files` 卷已还原且 `STORAGE_TYPE=local` |
| 启动报镜像拉不到 | `WEKNORA_VERSION` 指定的 tag 不存在 | 与旧服务器对齐版本，或 `docker compose pull` 可用版本 |

---

## 9. 回滚方案

若新服务器验证不通过、需回退：

- **新服务器数据未写入新数据**：直接 `docker compose down`，保留卷；排查后重新按 §6 恢复。
- **想回到旧服务器**：旧服务器停机后卷未删，直接 `docker compose up -d` 即可原样恢复服务（前提是旧机器磁盘仍在）。
- **数据已污染需重来**：`docker compose down -v` 删除容器与卷（⚠️ 会清掉新服务器上的卷），再重新从 §6 恢复备份。

---

## 10. 最小命令速查（默认部署，无 profile）

旧服务器：
```bash
cd /path/to/WeKnora
docker compose down --remove-orphans
cd /tmp && mkdir -p wk-backup && cd wk-backup
for v in WeKnora_postgres-data WeKnora_data-files WeKnora_docreader-tmp; do
  docker run --rm -v "$v":/volume -v "$(pwd)":/backup busybox tar czf /backup/"$v".tar.gz -C /volume .
done
cp /path/to/WeKnora/.env /path/to/WeKnora/config/config.yaml /tmp/wk-backup/
tar czf /tmp/wk-migration.tar.gz -C /tmp/wk-backup .
scp /tmp/wk-migration.tar.gz user@新服务器:/tmp/
```

新服务器（目录名保持 `WeKnora`）：
```bash
mkdir -p /opt/WeKnora && cd /opt/WeKnora   # 放入相同版本的项目代码
tar xzf /tmp/wk-migration.tar.gz -C /tmp/ && cd /tmp/wk-backup
cp .env /opt/WeKnora/.env && cp config.yaml /opt/WeKnora/config/config.yaml
for v in WeKnora_postgres-data WeKnora_data-files WeKnora_docreader-tmp; do
  docker volume create "$v"
  docker run --rm -v "$v":/volume -v /tmp/wk-backup:/backup busybox tar xzf /backup/"$v".tar.gz -C /volume
done
cd /opt/WeKnora && docker compose up -d
curl -f http://localhost:8080/health
```

---

*本手册聚焦「单机 Docker 全量迁移」。若你的部署涉及外部托管数据库 / 对象存储 / 多机，或需要零停机迁移，需另行评估，不在本文范围。*
