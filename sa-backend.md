# Phase0.5 DevB 需求文档（给 AI Agent）

## 项目目录结构（与本任务相关）

```text
sales-agent/
├─ backend/
│  ├─ main.py                          # FastAPI入口，日志初始化
│  ├─ api/
│  │  ├─ chat.py                       # 对话SSE入口（chat -> wrapper 起点）
│  │  └─ debug.py                      # 调试触发入口（也会调用 wrapper）
│  ├─ agent/
│  │  └─ wrapper.py                    # Agent主链路（state/scheduler/sdk/tool）
│  ├─ scheduler/
│  │  └─ trigger.py                    # 规则匹配与goal判定
│  ├─ tools/                           # 工具执行日志（含状态写入）
│  ├─ middleware/
│  │  ├─ auth.py
│  │  └─ audit.py
│  └─ context/
│     └─ brand_context.py              # 现有contextvars，仅brand/line/mode
├─ docs/service-transformation/
│  └─ devs.md                          # 三人分工与Gate规则
└─ SPEC.md                             # 本文档
```

## 当前状况描述（基于已读代码）
1. `devs.md` 对 Phase0.5 的 DevB 任务很明确：补齐 `trace_id/goal_id/user_id` 日志字段，并产出“日志字段字典”。
2. 当前后端日志在 `backend/main.py` 使用 `logging.basicConfig` 普通文本格式，未统一注入 `trace_id/goal_id/user_id`。
3. 请求主链路是 `api/chat.py -> agent/wrapper.py -> scheduler/trigger.py -> tools/*`，日志大量存在，但字段不统一，跨模块串联困难。
4. 当前仅有 `brand_context.py`，没有请求级 `trace_id/user_id/goal_id` 上下文容器。
5. `audit.py` 有 `request_id`，但属于单独审计文件，不是业务主链路日志，且与 wrapper/tool 日志未打通。

## 任务目标（Phase0.5 / DevB）

### 简述
1. 在请求入口生成并传播 `trace_id`（支持透传已有请求头）。
2. 在关键节点统一输出 `goal_id`（无 goal 时给出约定值，如 `passive_response` 或 `-`）。
3. 在关键日志统一带上 `user_id`。
4. 产出“日志字段字典”文档，作为 Gate0 交付物之一。

### 需求原因
1. 让 SOP 问题可复现、可定位、可对照。
2. 目前同一请求跨 `chat -> wrapper -> scheduler -> tools` 的日志链路断裂。
3. Gate0 的“场景回放 + 日志对照”会直接依赖这批字段。

## 范围边界

### In Scope（本次必须做）
1. 日志字段标准化（`trace_id/goal_id/user_id`）及链路传播。
2. 主链路关键节点日志补齐。
3. 字段字典文档落地。
4. 最小必要测试（上下文传播 + 关键日志字段存在性）。

### Out of Scope（本次不做）
1. 不改业务规则与SOP判定逻辑。
2. 不引入数据库、Redis、队列等架构升级。
3. 不做日志平台改造（ELK/Prometheus/Grafana）和告警平台建设。
4. 不做全量日志重构，仅覆盖“关键节点”。

## 需要修改的文件和修改内容（要求）

### A. 上下文与日志基础设施
1. `backend/context/`：新增请求链路上下文模块（建议 `request_context.py`）。
   - 提供 `trace_id/user_id/goal_id` 的 contextvars 存取函数与作用域管理。
   - 与现有 `brand_context.py` 并存，保持职责清晰。
2. `backend/main.py`：
   - 挂载日志上下文注入机制（Filter/Adapter 二选一，优先标准库实现）。
   - 调整日志格式，确保每条业务日志可打印 `trace_id/user_id/goal_id`（缺失时显示 `-`，避免 KeyError）。
3. `backend/middleware/`：新增 trace 中间件（建议 `trace.py`）。
   - 入口生成 `trace_id`（优先读取请求头 `X-Trace-ID`，否则生成 UUID）。
   - 写入 contextvars，并在响应头回传 `X-Trace-ID`。

### B. 请求入口与主链路打点
1. `backend/api/chat.py`：
   - 在 `/stream` 请求入口绑定 `user_id` 到上下文。
   - 解析 SOP goal 后绑定 `goal_id`。
   - 关键日志（收到请求、goal解析、事件异常）必须带三字段。
2. `backend/agent/wrapper.py`：
   - `process_message` 开始时统一设置/刷新 `goal_id`（goal为空时使用约定默认值）。
   - SDK 调用开始、结果、异常、SOP初始化等关键日志必须带三字段。
3. `backend/scheduler/trigger.py`：
   - 规则命中日志统一带 `goal_id`。
   - 若无命中，关键debug日志至少保留 `trace_id/user_id`。

### C. 工具层关键日志补齐（只改关键写状态工具）
1. `backend/tools/tags.py`
2. `backend/tools/order_link.py`
3. `backend/tools/sop_state.py`
4. `backend/tools/send_message.py`
5. `backend/tools/transfer_human.py`
6. `backend/tools/scheduler.py`
7. `backend/tools/youzan.py`

要求：成功/失败日志至少带 `user_id`；如当前语义关联 goal，则同时带 `goal_id`。

### D. 日志字段字典（必须新增）
新增文档：`docs/service-transformation/log-field-dictionary-phase0.5.md`（文件名可微调，但必须在该目录）。

最少包含以下字段定义：
1. `trace_id`
2. `user_id`
3. `goal_id`
4. `brand_id`
5. `line_id`
6. `session_id`（若存在）
7. `event`（日志事件名）
8. `status`（ok/error）
9. `error_code`（可选）

每个字段必须写清：含义、类型、来源、是否必填、示例、缺失时策略。

## 验收标准（Gate0 对齐口径）
1. 同一请求在 `chat -> wrapper -> scheduler -> tools` 至少可通过 `trace_id` 串起来。
2. 关键节点日志可直接检索到 `user_id`。
3. 有 goal 的链路均可检索 `goal_id`（无 goal 的链路按约定值处理）。
4. 响应头返回 `X-Trace-ID`，可与日志对应。
5. `docs/service-transformation/log-field-dictionary-phase0.5.md` 完整可读。

## 测试要求（最小集合）
1. 新增/更新测试，覆盖：
   - contextvars 在并发场景下不会串号（trace/user/goal）。
   - `/api/chat/stream` 响应包含 `X-Trace-ID`。
   - 至少一个主链路日志断言包含三字段。
2. 至少运行并记录：
   - `pytest -q backend/tests/test_contextvars_isolation.py`
   - 与本次新增测试对应的 pytest 命令

## 非功能约束
1. 默认不新增外部依赖，优先 Python 标准库实现。
2. 不能破坏现有 API 请求/响应结构（除新增响应头 `X-Trace-ID`）。
3. 不能影响现有 SOP 行为与状态写入语义。
4. 日志字段缺失必须有安全默认值，禁止因日志格式导致运行时异常。

## 通用限制

### 文件相关限制
1. 先确认目录结构，再定位文件，禁止拍脑袋改。
2. 本任务主战场是 `backend/`，但 `docs/service-transformation/` 交付物必须同步落地。
3. 和任务相关的文件必须阅读后再改，禁止只靠猜测。

### 流程限制
1. 在开始改代码前，先输出你的理解和修改计划，等我确认后再动手。
2. 完成后更新根目录 `versions.md`（不存在则新建），按 `v1/v2/v2.1` 递增记录。
3. 产出一段简短 git 提交说明（中文，1-3行）。

## checklist（用于修改后对照，必须全部 OK）
- [ ] 已完整阅读本任务相关文件（至少包含 chat/wrapper/trigger/关键tools/main/context/devs）
- [ ] 已实现 `trace_id` 入口生成与全链路传播
- [ ] 已实现关键节点 `goal_id` 统一输出
- [ ] 已实现关键日志 `user_id` 统一输出
- [ ] 已新增并填写日志字段字典文档
- [ ] 已补充并执行最小必要测试
- [ ] 已更新 `versions.md`
- [ ] 已提供 git 提交说明
