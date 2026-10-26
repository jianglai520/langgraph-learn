# Docker + PostgreSQL 持久化环境（实测记录）

> 检查时间：2026-10-26。以下结果来自实际执行 `docker` 与 `psql` 命令，不是推测。

## 一、结论：`langgraph_db` 已经建好了

| 检查项 | 实际结果 |
|---|---|
| 容器名 | `langgraph-postgres` |
| 镜像 | `postgres:16`（服务端 16.15） |
| 状态 | Up，端口 `0.0.0.0:5432 -> 5432` |
| 数据库 | `langgraph_db` ✅ 已存在，owner 是 `langgraph_user` |
| 其他库 | `postgres`、`template0`、`template1` |
| 字符集 | UTF8，locale `en_US.utf8` |
| `public` 下的表 | **1 张**：`messages`（你自己建的消息表，已写入 2 行） |
| LangGraph 的 checkpoint 表 | **尚未创建**（`checkpoints` 等 4 张表不存在） |
| 数据卷 | 匿名卷 `a935a9fa…`（不是命名卷） |
| 重启策略 | `no`（Docker 重启后容器不会自启） |

也就是说：**库有了，消息表也有了，但 LangGraph 自带的 checkpointer 表还没建。**

这里要分清两条完全不同的持久化路线：

- `messages` 表：你自己写 SQL 存对话记录，跟 LangGraph 无关
- `checkpoints` 等 4 张表：`PostgresSaver.setup()` 自动创建，供 LangGraph 的 checkpointer 使用

两者可以共存，互不冲突。

## 二、连接串

LangGraph 里用这个：

```
postgresql://langgraph_user:123456@127.0.0.1:5432/langgraph_db
```

- 用户名：`langgraph_user`
- 密码：`123456`
- 注意用 `127.0.0.1`，不要用 `localhost`。本机实测 `localhost` 会卡住（解析到 IPv6 后握手不返回），换 `127.0.0.1` 秒连。

### 为什么 `localhost` 会卡

Windows 上 `localhost` 同时解析出 `::1` 和 `127.0.0.1`，`::1` 排在前面。Docker 把端口发布到了 `0.0.0.0` 和 `[::]` 两个地址，但这台机器上 IPv6 那条转发不通（实测 6 秒超时），客户端就堵在第一个地址上。

三个处理办法：

1. 连接串统一写 `127.0.0.1`（首选，零成本）
2. 让容器只发布 IPv4：`-p 127.0.0.1:5432:5432`。`::1` 会立刻拒绝连接，客户端自动回落到 IPv4，`localhost` 也能用（需要重建容器，数据卷保留即可）
3. 连接串加 `?connect_timeout=3`，至少失败得快，不会一挂几分钟

## 三、Docker 日常操作

```powershell
# 看所有容器（含已停止的）
docker ps -a

# 只看这一个
docker ps -a --filter "name=langgraph-postgres"

# 启停重启
docker start langgraph-postgres
docker stop langgraph-postgres
docker restart langgraph-postgres

# 看日志（启动失败先看这个）
docker logs -f langgraph-postgres

# 进容器内部
docker exec -it langgraph-postgres bash

# 直接进 psql 交互界面（最常用）
docker exec -it -e PGPASSWORD=123456 langgraph-postgres psql -U langgraph_user -d langgraph_db
```

⚠️ 你的宿主机**没有安装 psql**，所以所有 psql 操作都要走 `docker exec`。

## 四、psql 里最常用的命令

进去以后（提示符变成 `langgraph_db=#`）：

| 命令 | 作用 |
|---|---|
| `\l` | 列出所有数据库 |
| `\c langgraph_db` | 切换到某个库 |
| `\dt` | 列出当前库的表 |
| `\d 表名` | 看表结构 |
| `\du` | 列出所有用户/角色 |
| `\dn` | 列出 schema |
| `select version();` | 看服务端版本 |
| `\q` | 退出 |

建库 / 删库（要在 `postgres` 库里执行）：

```bash
# 新建
docker exec -it -e PGPASSWORD=123456 langgraph-postgres \
  psql -U langgraph_user -d postgres -c "CREATE DATABASE demo_db"

# 删除（当前连着的库删不掉，先 \c postgres 切走）
docker exec -it -e PGPASSWORD=123456 langgraph-postgres \
  psql -U langgraph_user -d postgres -c "DROP DATABASE demo_db"
```

## 五、输出太长卡住时：你在 less 分页器里

查出来的行数多了，psql 会自动调用 `less` 分页，屏幕底部出现 `(END)`，上面一堆 `~`。这不是卡死。

| 按键 | 作用 |
|---|---|
| `q` | 退出分页器，回到 `langgraph_db=#` |
| 空格 | 下一屏 |
| `b` | 上一屏 |
| `Ctrl+C` | **没用**，只会把 `Cancel request sent` 发给数据库，分页器不退出 |

永久关掉分页，两种方式：

```sql
-- 当前会话有效
\pset pager off
```

```bash
# 启动时就关掉，输出直接刷屏
docker exec -it -e PGPASSWORD=123456 -e PSQL_PAGER=cat langgraph-postgres \
  psql -U langgraph_user -d langgraph_db
```

非交互方式（脚本、本文件里的命令）可以加 `-P pager=off`。

## 六、当前库里的数据

`messages` 表是你自己建的，结构如下：

```
 Column     | Type                        | Nullable | Default
------------+-----------------------------+----------+---------------------------
 id         | integer                     | not null | nextval('messages_id_seq')
 thread_id  | text                        | not null |
 role       | text                        | not null |
 content    | text                        | not null |
 created_at | timestamp without time zone |          | CURRENT_TIMESTAMP
Indexes: messages_pkey PRIMARY KEY (id)
```

现有 2 行（thread `chapter_6_6.2.4`）：

| id | role | content | created_at |
|---|---|---|---|
| 1 | user | 你好，我是老王 | 2026-10-26 05:28:52.119075 |
| 2 | assistant | 老王你好！有什么可以帮你的？ | 2026-10-26 05:28:52.122186 |

两条记录只差 3 毫秒，说明写入流程是通的。

常用查询：

```sql
-- 看某个会话的全部消息
select * from messages where thread_id = 'chapter_6_6.2.4' order by id;

-- 看有多少个会话
select thread_id, count(*) from messages group by thread_id;
```

## 七、让 LangGraph 用上这个库

`langgraph-checkpoint-postgres` 已经装在你的环境里（3.0.5，配 `psycopg` 3.3.3）。第一次用必须先建表：

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://langgraph_user:123456@127.0.0.1:5432/langgraph_db"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()          # 建表，可重复执行，不会报错
    graph = builder.compile(checkpointer=checkpointer)
    result = graph.invoke(
        {"notes": []},
        {"configurable": {"thread_id": "demo-1"}},
    )
print(result)
```

异步版本：

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:
    await checkpointer.setup()
```

`setup()` 会创建 4 张表：

- `checkpoints`
- `checkpoint_blobs`
- `checkpoint_writes`
- `checkpoint_migrations`

跑完之后用 `\dt` 就能看到它们。**不跑 `setup()` 直接用**，会得到 `relation "checkpoints" does not exist` 之类的报错。

## 八、从零重建这套环境

如果你想重新来一遍，完整命令是：

```bash
docker run -d \
  --name langgraph-postgres \
  -e POSTGRES_USER=langgraph_user \
  -e POSTGRES_PASSWORD=123456 \
  -e POSTGRES_DB=langgraph_db \
  -p 5432:5432 \
  -v langgraph_pgdata:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:16
```

和你当前容器的两点差别，建议顺手改掉：

1. **命名卷**：`-v langgraph_pgdata:/var/lib/postgresql/data`。你现在是匿名卷，名字是一串哈希，将来不敢删也不敢认。
2. **`--restart unless-stopped`**：你现在是 `no`，电脑重启后容器不会自己起来，连接就会失败。

## 九、常见报错对照

| 报错 | 原因 | 处理 |
|---|---|---|
| `role "postgres" does not exist` | 容器初始化时 `POSTGRES_USER` 设成了 `langgraph_user`，没有 `postgres` 这个角色 | 一律用 `-U langgraph_user` |
| `password authentication failed` | 没带 `PGPASSWORD` 或密码不对 | `-e PGPASSWORD=123456` |
| 连接一直卡住不返回 | 用了 `localhost`，走 IPv6 卡住 | 换成 `127.0.0.1` |
| `port is already allocated` | 5432 被别的容器占用 | `docker ps -a` 找到占用者并停止 |
| `relation "checkpoints" does not exist` | 没执行 `setup()` | 先 `checkpointer.setup()` |
| 屏幕底部出现 `(END)`，敲什么都在闪 | 进了 less 分页器 | 按 `q` 退出；`Ctrl+C` 无效 |
| 容器删了，数据还在吗 | 数据在卷里，删容器不删卷 | 别乱删卷；建议改命名卷 |

顺带一提：你还有一个 `pg-langchain`（postgres:16，7 天前退出，未映射端口）和 `milvus-standalone`（6 天前异常退出）。它们和当前库无关，但容易混淆，必要时清理。

## 十、备份与恢复

```bash
# 备份
docker exec -e PGPASSWORD=123456 langgraph-postgres \
  pg_dump -U langgraph_user -d langgraph_db > backup.sql

# 恢复
docker exec -i -e PGPASSWORD=123456 langgraph-postgres \
  psql -U langgraph_user -d langgraph_db < backup.sql
```

## 十一、自检清单

- [x] 容器 `langgraph-postgres` 在跑
- [x] 数据库 `langgraph_db` 存在
- [x] 从宿主机 `127.0.0.1:5432` 能连上
- [x] `messages` 表写入成功（thread `chapter_6_6.2.4`，2 行）
- [ ] 执行过 `checkpointer.setup()`，`\dt` 能看到 4 张 checkpoint 表
- [ ] 容器改成命名卷 + `--restart unless-stopped`
- [ ] `.env` 里的连接串用的是 `127.0.0.1`
