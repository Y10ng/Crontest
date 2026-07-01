## 快速开始

### 依赖

- macOS 或 Linux
- Python 3.12+
- `uv`
- Docker，使用 worker container 或 docker compose 时需要
- Node.js / npm，仅在修改前端源码并重新构建静态资源时需要

### 准备 Worker 镜像

默认配置使用本地镜像 `cairn-worker-container:local`。在当前机器架构上构建：

```bash
docker build -f container/Dockerfile . -t cairn-worker-container:local
```

查看镜像平台：

```bash
docker image inspect cairn-worker-container:local --format 'os={{.Os}} arch={{.Architecture}}'
```

如果你确实要按 `linux/amd64` 运行，则需要这样构建：

```bash
docker buildx build --platform linux/amd64 -f container/Dockerfile . -t cairn-worker-container:local --load
```

### 配置 Dispatcher

Dispatcher 运行配置由本地 `dispatch.yaml` 承载，YAML 是 runtime、tasks、container、images、execution_profiles、workers、common_env、server 和 dispatcher_id 的唯一真相源。可以从样例复制后编辑：

```bash
cp dispatch.yaml.example dispatch.yaml
```

Console 的「Dispatcher」页默认使用 dispatcher 启动时上报的 `config_path`，提供表单编辑与原始 YAML 两种入口；如果 Server 与 Dispatcher 路径不一致，也可以在 YAML 中设置 `server_reachable_path` 覆盖。Server 无法读取该路径时，Console 会展示 dispatcher 心跳上报快照和只读 YAML 视图。

### 本地启动

启动 Server：

```bash
uv run --project cairn cairn serve
```

打开 UI：

```text
http://127.0.0.1:8000
```

先运行 startup healthcheck：

```bash
uv run --project cairn cairn dispatch --config dispatch.yaml --startup-healthcheck-only
```

启动 Dispatcher：

```bash
uv run --project cairn cairn dispatch --config dispatch.yaml
```

只跑一轮调度，适合调试：

```bash
uv run --project cairn cairn dispatch --config dispatch.yaml --once
```

### Docker Compose

Compose 会启动 `cairn-server` 和 `cairn-dispatcher`，并把数据挂载到 `./datas/cairn/`：

```bash
docker compose up --build
```

注意：Compose 使用 `dispatch.compose.yaml` 挂载到 Server 与 Dispatcher 的 `/etc/cairn/dispatch.yaml`。Compose 只构建 Cairn 应用容器，不会自动构建 YAML 里指定的 worker 镜像，worker 镜像仍需提前准备。

## Dispatcher 运维说明

`dispatch.yaml` 是 dispatcher 引导与运行配置真相源。Server 不保存明文 worker env；`/dispatcher/yaml` 与 `/dispatcher/config` 默认读写 dispatcher 上报的启动 `config_path`，`server_reachable_path` 仅作为路径覆盖项。目标路径仍需位于允许根内，保存前统一走 dispatcher 配置校验。

推荐变更流程：

1. 编辑 dispatcher 正在读取的 `dispatch.yaml`。
2. 如果新增镜像，先 `docker build` 或 `docker pull`，并用 `docker image inspect` 确认镜像可运行。
3. 运行启动健康检查：

```bash
uv run --project cairn cairn dispatch --config dispatch.yaml --startup-healthcheck-only
```

4. 已运行 dispatcher 会按 YAML mtime/sha256 轮询热加载；需要突破 `hard_cap` 时重启 dispatcher 并调整 `CAIRN_DISPATCHER_HARD_CAP`。
5. 在 Server 侧确认当前 dispatcher 上报快照：

```bash
curl -sS http://127.0.0.1:8000/workers
curl -sS http://127.0.0.1:8000/images
curl -sS http://127.0.0.1:8000/capabilities
```

Worker 运维要点：

- `workers[].name` 必须唯一；`priority` 数值越小越优先；`max_running` 是单个 worker 的并发上限。
- `common_env` 会合并到每个 worker 的 `env`，worker 自己的同名 key 优先。
- 必需环境变量由 worker 类型决定：`claudecode` 需要 `ANTHROPIC_MODEL / ANTHROPIC_BASE_URL / ANTHROPIC_AUTH_TOKEN`，`codex` 需要 `CODEX_MODEL / CODEX_BASE_URL / OPENAI_API_KEY`，`pi` 需要 `PI_MODEL / PI_BASE_URL / PI_API_KEY / PI_PROVIDER_API`。
- 心跳只上报 worker env key 列表，不上报 env 值；明文密钥只留在 dispatcher 本机 YAML 中。

镜像、capability 与 execution profile 要点：

- `images[]` 声明可被 profile 引用的容器镜像，`container.image` 仍作为旧配置和默认兜底。
- `container.cap_add` 是 `execution_profiles[].cap_add` 的运行时基线。profile 只能选择该基线内的 capability。
- 场景只选择 `profile_id`；profile 决定容器镜像、网络、capability 与隔离策略，worker 路由完全由 `workers[].task_types / priority / max_running` 决定。
- capability 只应按最小权限添加，尤其谨慎使用 `NET_ADMIN`、`SYS_ADMIN` 等高权限项。

运行期注意事项：

- `container.execution_profile` 是旧兼容字段；新配置使用 `execution_profiles[].isolation`。若场景 profile 对应的 image、network、capability 等运行时签名发生变化，正在运行的项目容器不会在任务中途被强制替换；等项目空闲、容器停止或被清理后，新配置才会创建新容器。
- `completed_action: stop` 会保留完成后的容器便于排查；`remove` 更干净，但会删除运行后的容器状态。
- `runtime.max_workers` 可热更新，但不能超过 dispatcher 进程启动时的 `hard_cap`；默认 `hard_cap=64`。
- 心跳用于存活、监控、场景选项和保存期校验，`worker_pool`、`container_images`、`cap_baseline` 与 `declared_inventory` 是 dispatcher 已加载 YAML 的回显。
- 场景不再保存 worker 选择；dispatcher 运行时只按 worker 启用状态、`task_types`、优先级、并发、冷却和拒绝窗口选择 worker。

## 创建项目

推荐直接在 UI 中创建项目。也可以使用 API：

```bash
curl -sS http://127.0.0.1:8000/projects \
  -H 'content-type: application/json' \
  -d '{
    "title": "本地测试项目",
    "origin": "起点事实：这里描述已知输入、授权范围或初始环境。",
    "goal": "目标事实：这里描述完成条件。",
    "hints": [
      {"content": "优先走低风险验证路径。", "creator": "human"}
    ],
    "targets": [
      {"type": "domain", "value": "example.com"}
    ]
  }'
```

项目创建后，Dispatcher 会在调度循环中选择可执行任务并派发给 Worker。Monitor UI 会展示运行任务、Worker 健康、错误聚合、artifact、result audit 和 Claude Code hook 操作摘要。

## 审计数据

Server 默认使用 SQLite。直接本地运行时默认路径为：

```text
~/.local/share/cairn/cairn.db
~/.local/share/cairn/artifacts/
```

Docker Compose 默认挂载到：

```text
./datas/cairn/
```

当前审计主线：

- `TaskRun`：任务生命周期、Worker、session、model、prompt group、配置 hash、容器信息、开始/结束时间和错误类别。
- `stdout/stderr artifact`：完整原始输出保存为 artifact，不依赖 LLM 总结。
- `declared artifact`：Worker 在协议 JSON 中声明的报告、日志、截图等文件。
- `hook_log artifact`：Claude Code hooks 写出的原始 JSONL。
- `hook_operation event`：从 hook JSONL 中提取的脱敏操作摘要。
- `validation_result` 和 `protocol_write`：Dispatcher 对 Worker 输出的校验和协议写入记录。
- `FactEvidence`：Fact / Complete 与 artifact 的绑定关系。

轻量知识库位于：

```text
datas/cairn/knowledge/
```

它是手工维护的文件型知识库。Worker prompt 会提示 Agent 只在相关时用 `rg`、`ls` 和定向读取，不会把它当作 RAG 或向量数据库。

Console 的「知识库」页可以创建/删除知识集、上传或编辑 `.md/.yaml/.yml/.json/.txt` 文件，并维护 Markdown front matter（`title/tags/applies_to/summary/last_verified_at`）。场景配置里的 `knowledge` 只能引用真实存在的知识集；被场景引用的知识集不能删除。默认根路径可通过 `CAIRN_KNOWLEDGE_ROOT` 覆盖，server 与 dispatcher 会使用同一位置。

Prompt 模板组默认位于：

```text
datas/cairn/prompts/
```

Console 的「Prompt 模板组」页可以从内置组克隆、新建、编辑和删除受管模板组。每组必须包含 `reason.md / explore.md / explore_conclude.md / bootstrap.md / bootstrap_conclude.md`，保存时会校验必需占位符；可选 `_output_lang.md`，没有时会继承内置 `default` 的输出语言片段。内置模板组保留为只读基线，受管目录中的同名组会覆盖内置组；被场景引用的受管组不能删除。默认根路径可通过 `CAIRN_PROMPTS_ROOT` 覆盖，server 与 dispatcher 会使用同一位置。

场景配置只保留 Prompt 组、知识库和 `profile_id` 等策略引用，不再包含 `enabled_workers` / `worker_ids`。`/workers`、`/images`、`/capabilities` 提供只读清单，数据来自 `dispatcher_status` 心跳快照；Dispatcher 页面通过 `/dispatcher/config` 表单或 `/dispatcher/yaml` 原文编辑同一份 `dispatch.yaml`。

## 开发

运行后端测试：

```bash
uv run --project cairn pytest
```

前端源码位于 `cairn/frontend/`，构建产物输出到 FastAPI 静态目录 `cairn/src/cairn/server/static/`：

```bash
cd cairn/frontend
npm install
npm run build
```

主要文档：

- `docs/specs/stable-agent-platform.md`
- `docs/specs/dispatcher-design.md`
- `docs/specs/server-protocol.md`
- `docs/progress/stable-agent-platform-progress.md`
- `docs/progress/modules/core-protocol-progress.md`
- `docs/progress/modules/dispatcher-worker-progress.md`
- `docs/progress/modules/audit-artifact-progress.md`
- `docs/progress/modules/claude-hooks-progress.md`
- `docs/progress/modules/targets-policy-progress.md`
- `docs/progress/modules/monitor-ui-progress.md`
- `docs/progress/code-reduction-plan.md`

修改 Server 协议、Dispatcher 行为、Worker adapter contract、TaskRun / AgentEvent / Artifact / Evidence 模型、Claude Code hooks、Target 语义或 Monitor UI 主要交互时，请同步更新对应文档。
