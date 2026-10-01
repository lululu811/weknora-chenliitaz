# 部署与排障手册

面向第一次部署本仓库的用户。**遇到问题先跑 `./scripts/doctor.sh`**，它会直接告诉你卡在哪。

---

## 0. 最短路径

```bash
cp .env.example .env
./scripts/doctor.sh      # 自检
docker compose up -d
open http://localhost
```

---

## 1. 端口占用

默认占用：

| 端口 | 服务 | 改法（写在 `.env`） |
|---|---|---|
| 80 | `frontend`（Web 界面） | `FRONTEND_PORT=8081` |
| 8080 | `app`（后端 API） | `APP_PORT=8090` |
| 50051 | `docreader` | 默认仅容器内暴露，通常不冲突 |
| 50052 | `python-service` | 仅 finance profile 需要 |

> ⚠️ 改 `FRONTEND_PORT` 后，浏览器访问地址也要跟着改（`http://localhost:8081`）。

**改端口时注意**：Web 界面前端是通过反向代理访问后端的。改 `APP_PORT` 只改宿主机映射，一般无需改其他配置。

---

## 2. 常见启动失败

### 2.1 `Bind for 0.0.0.0:80 failed: port is already in use`

80 端口被占用——通常是本机已装 nginx / Apache，或旧容器没停。

```bash
lsof -i :80                    # 看谁占着
docker compose down            # 停掉本项目容器
# 或在 .env 里改：FRONTEND_PORT=8081
```

### 2.2 容器反复重启 / `healthcheck failed`

看具体日志，不要猜：

```bash
docker compose ps                       # 看哪个服务不健康
docker compose logs --tail=100 app      # 后端
docker compose logs --tail=100 postgres # 数据库
```

**app 启动慢是正常的**（首次要跑数据库迁移，healthcheck 的 `start_period` 是 60 秒）。耐心等 1-2 分钟。

### 2.3 `could not find .env`

```bash
cp .env.example .env
```

### 2.4 首次启动后打不开页面，但容器都在跑

多半是模型没配。打开 `http://localhost` 按引导走，**在 Web 界面里配置对话模型和向量模型**（不需要手改配置文件）。

若用本地 Ollama，先确认它起来了：

```bash
ollama serve
curl http://127.0.0.1:11434/api/tags
```

---

## 3. 机器要求与常见瓶颈

| 项 | 建议 | 说明 |
|---|---|---|
| 内存 | ≥ 8 GB | 默认栈（5 服务）约需 3-4 GB |
| 磁盘 | ≥ 10 GB | 加上模型和文档会持续增长 |
| 架构 | x86_64 / arm64 均可 | 用 `--profile full` 拉 Milvus/Doris 时，arm64（Apple Silicon）镜像可能不完整，建议用 x86_64 机器 |

内存不够的典型表现：容器无报错直接消失，或宿主机整体卡死。

```bash
docker stats --no-stream      # 看谁在吃内存
```

减负手段：
- 不用 `--profile full`，只用默认 5 服务
- 向量库用 `RETRIEVE_DRIVER=sqlite`（默认，内存占用最低）
- 关掉沙箱：`WEKNORA_SANDBOX_MODE=disabled`
- 关掉图谱：`ENABLE_GRAPH_RAG=false`

---

## 4. 投研栈（finance profile）

### 4.1 工具返回「底层数据源不可用」

**这是预期行为，不是故障。** 表示 `HITHINK_DB_DIR` 没配或目录里没有 `.duckdb` 文件。

```bash
export HITHINK_DB_DIR=/path/to/your/hithink-db
ls "$HITHINK_DB_DIR"/*.duckdb   # 确认文件在
docker compose --profile finance up -d
```

`HITHINK_DB_DIR` 写进 `.env` 也可以，但要记住它指向**宿主机的绝对路径**（容器内统一挂到 `/data/hithink`）。

### 4.2 容器内看不到数据

Docker Desktop 需允许访问该目录。路径在容器内恒为 `/data/hithink`，验证：

```bash
docker compose --profile finance exec python-service ls -la /data/hithink
```

### 4.3 `601059.SH` 之类代码查不到

部分标的可能无数据或已退市。工具会明确告诉你「该股票无数据」，属正常。

---

## 5. 数据与备份

默认部署的数据落在这里：

| 内容 | 位置 |
|---|---|
| 数据库 | Docker 卷 `postgres-data` |
| 上传的文件 | Docker 卷 `data-files` |

```bash
# 备份
docker compose exec postgres pg_dump -U <user> weknora > backup.sql

# 查看卷
docker volume ls | grep weknora
```

**注意**：升级或改 compose 文件后，旧数据在卷里，不会丢；但 `docker compose down -v` 会**删掉卷，数据不可恢复**。

---

## 6. 彻底重置

```bash
docker compose down          # 停服务，保留数据
docker compose down -v       # ⚠️ 连数据一起删
docker compose up -d         # 重新拉起并跑迁移
```

---

## 7. 升级

```bash
git pull
docker compose pull          # 拉新镜像
docker compose up -d         # 重建容器
```

上游发布说明见 https://weknora.weixin.qq.com/docs/07-releases 。**跨大版本升级前建议先备份数据库**（见第 5 节）。

本 fork 用 `upstream` 远端同步上游：

```bash
git remote -v                 # 确认 upstream 是只读的
git fetch upstream && git rebase upstream/main
```

---

## 8. 还是解决不了

带上这些信息提问，效率最高：

```bash
docker compose version
docker version --format '{{.Server.Version}}'
docker compose ps
docker compose logs --tail=200 app
uname -a
```

以及 `doctor.sh` 的完整输出。
