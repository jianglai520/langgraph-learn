# LangGraph 学习笔记

按 LangGraph 官方教程顺序动手实践的一套笔记：**5 章、81 个 notebook**，每节一个主题，都可以单独运行。

内容覆盖从「状态怎么定义」到「持久化、人机协同、流式输出、子图与设计模式」的完整路径，代码以能跑通为准，不追求抽象讲解。

## 快速开始

```bash
# 1. 准备环境（Python 3.13）
conda activate langgraph

# 2. 安装依赖
pip install -r requirements.txt

# 3. 验证环境
# 打开 test.ipynb，它会打印 Python 与 LangGraph 的版本
```

> `requirements.txt` 锁的是完整版本。环境已经能跑的话不要再执行它，否则会把现有包对齐（含降级）到清单版本；只想补装缺的包，单独 `pip install 包名` 更安全。

然后在项目根目录建一个 `.env`（**不要提交**）：

```
DEEPSEEK_API_KEY=你的密钥
DEEPSEEK_BASE_URL=https://api.deepseek.com
```

笔记里用 `load_dotenv()` 读取这两个变量，模型统一走 DeepSeek（`deepseek-flash` / `deepseek-v4-flash`）。

## 目录结构

```
langgraph-learn/
├── chapter01/        状态与基础图（10 节）
├── chapter02/        控制流：边、路由、循环、并发（16 节）
├── chapter03/        持久化：检查点、历史、回放、Store（12 节）
├── chapter04/        人机协同 HITL：中断、审批、断点（17 节）
├── chapter05/        流式输出、子图、设计模式（26 节）
├── hitl_demo/        可独立启动的 Agent 示例（langgraph dev）
├── test.ipynb        环境自检
├── POSTGRES.md       Docker + PostgreSQL 环境实测记录
├── langgraph-superstep.html  superstep 概念可视化说明
└── requirements.txt  全量依赖
```

## 五章学习路径

| 章节 | 主题 | 节数 | 你会学到 |
|---|---|---|---|
| chapter01 | 状态与基础图 | 10 | TypedDict / dataclass / pydantic 三种状态定义、reducer 原理、`add_messages`、节点读写状态、输入与输出状态拆分 |
| chapter02 | 控制流 | 16 | `add_edge`、并行节点、条件路由、`Send` 动态分发、`Command`、fan-in、map-reduce、循环与退出、`retry`、`cache` |
| chapter03 | 持久化 | 12 | `InMemorySaver`、`PostgresSaver`、durability 模式、状态历史、回放与分叉、Store 长期记忆、运行上下文 |
| chapter04 | 人机协同 | 17 | `interrupt()`、审批与修改、工具审批、中断与检查点的关系、静态断点、`ToolNode` |
| chapter05 | 流式与子图 | 26 | 7 种 `stream_mode`、`astream_events`、子图的四种用法、五种设计模式、Agent 实现 |

## 各章覆盖清单

<details>
<summary><b>chapter01 · 状态与基础图</b></summary>

- 01 `TypedDict` 定义状态（后续统一用这个）
- 02 `dataclass` 定义状态
- 03 `pydantic` 定义状态
- 04 手写 reducer，理解合并规则
- 05 `add_messages` 消息累加
- 06 节点内读写状态（`Annotated[list, add]`）
- 07 `Overwrite` 强制覆盖写
- 08 节点读取状态
- 09 `InputState` / `OutputState` / 全局状态三分离
- 10 消息 + LLM 调用

</details>

<details>
<summary><b>chapter02 · 控制流</b></summary>

- 01 `add_edge` 基础连线
- 02 并行节点
- 03 条件路由
- 04 用 `path_map` 路由
- 05 不用 `path_map` 的路由写法
- 06 `defer`：节点延后到本轮末尾执行
- 07 动态分发 `Send`
- 08 `Command`：节点同时完成更新与跳转
- 09 fan-in：多个分支汇合
- 10 map-reduce
- 11 静态循环
- 12 用 `goto` 做循环
- 13 `remaining_steps` 剩余步数
- 14 循环的退出条件
- 15 `retry` 重试策略
- 16 `cache` 节点缓存

</details>

<details>
<summary><b>chapter03 · 持久化</b></summary>

- 01 `InMemorySaver` 内存检查点
- 02 `PostgresSaver` 落库（**依赖本机 Docker 里的 PostgreSQL**）
- 03 durability 三种写入模式
- 04 读取历史状态
- 05 异常场景演示
- 06 最终状态下的异常
- 07 修复异常后继续
- 08 回放（replay）
- 09 分叉（fork）
- 10 Store：跨会话长期记忆
- 11 在节点里用 Runtime 访问 Store
- 12 运行时上下文（context）

</details>

<details>
<summary><b>chapter04 · 人机协同 HITL</b></summary>

- 01 `interrupt()` 入门
- 02 并行分支下的中断
- 03 审批（approve）
- 04 审批并修改参数
- 05 工具调用审批
- 06 单节点中断
- 07 HITL 与检查点
- 08 并行 + 检查点
- 09 部分任务中断时的检查点
- 10 静态断点
- 11 静态断点作用于多节点
- 12 在 `invoke` 时设置断点
- 13 `tool_call` 手写处理
- 14 `ToolNode`
- 15 `ToolNode` + Runtime
- 16 包装 tool call
- 17 包装 + 缓存

</details>

<details>
<summary><b>chapter05 · 流式输出、子图与设计模式</b></summary>

- 01 `stream_mode="values"`
- 02 `stream_mode="updates"`
- 03 `stream_mode="messages"`
- 04 `stream_mode="checkpoints"`
- 05 `stream_mode="tasks"`
- 06 `stream_mode="debug"`
- 07 `stream_mode="custom"`
- 08 `astream_events`
- 09 子图作为函数调用
- 10 子图作为节点
- 11 子图 + 检查点
- 12 子图 + 中断
- 13 子图 + 内存
- 14 子图检查点进阶
- 15 多子图协作
- 16 多子图进阶
- 17 无状态子图
- 18 子图流式输出
- 19 子图内 LLM 流式
- 20 子图 `goto`
- 21 提示链（prompt chaining）
- 22 并行化（parallelization）
- 23 路由（routing）
- 24 编排者-工作者（orchestrator-worker）
- 25 评估-优化（evaluator-optimizer）
- 26 Agent

</details>

## PostgreSQL 持久化

chapter03 的 `02/05/06/07/08/10/11` 这几节用 PostgreSQL 存检查点。本机环境已完成配置：

| 项 | 值 |
|---|---|
| 容器 | `langgraph-postgres`（`postgres:16`），端口 `5432` |
| 数据库 | `langgraph_db` |
| 用户 / 密码 | `langgraph_user` / `123456` |
| 连接串 | `postgresql://langgraph_user:123456@127.0.0.1:5432/langgraph_db` |

第一次使用必须先建表：

```python
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()          # 幂等，可重复执行
```

会创建 4 张表：`checkpoints`、`checkpoint_blobs`、`checkpoint_writes`、`checkpoint_migrations`。

完整说明（Docker 命令、psql 用法、报错对照、备份恢复）见 **[POSTGRES.md](POSTGRES.md)**。

两个容易踩的点：

- 连接串用 `127.0.0.1`，不要用 `localhost`（本机 IPv6 转发不通，会卡住）
- 容器里没有 `postgres` 这个角色，`psql -U postgres` 会直接报错，要用 `-U langgraph_user`

## hitl_demo：可启动的 Agent 示例

`hitl_demo/` 是一个可以被 `langgraph dev` 加载的最小项目，用 `langgraph.json` 注册了两个图：

| 图 | 文件 | 内容 |
|---|---|---|
| `graph` | `src/agent.py` | 用 `interrupt()` 依次收集用户名、年龄、性别 |
| `chat_graph` | `src/chat_agent.py` | DeepSeek + 工具调用 + 中断审批的对话 Agent |

启动方式：

```bash
cd hitl_demo
langgraph dev          # 需要 langgraph-cli，已在 requirements.txt 里
```

启动后会在浏览器打开 Studio，可以点开图、单步执行、发送 `resume` 值来继续被中断的流程。

> 如果 `langgraph dev` 在安装依赖时报错，把 `langgraph.json` 里的 `"dependencies": ["."]` 改成 `[]`，或在 `hitl_demo/` 下补一个 `pyproject.toml`。原因是当前目录没有包描述文件，pip 无法以本地包的方式安装它。

## 补充材料

- **`POSTGRES.md`** —— Docker + PostgreSQL 的实测记录：容器信息、日常命令、psql 常用操作、6 条报错对照、备份恢复
- **`langgraph-superstep.html`** —— 用「跑一圈」的比喻解释 superstep：并行节点如何共享一步、状态在步边界如何提交，含本机真实输出
- **`test.ipynb`** —— 打印 Python 与 LangGraph 版本，换环境后先跑它

## 环境说明

- Python 3.13，conda 环境名为 `langgraph`
- 依赖清单以 `requirements.txt` 为准，除了 LangGraph 本体，还包含 LangChain、MCP、文档解析、向量库等组件
- `.env` 已在 `.gitignore` 中忽略；`__pycache__/`、`.ipynb_checkpoints/`、`.langgraph_api/`、`.idea/` 同样不入库
