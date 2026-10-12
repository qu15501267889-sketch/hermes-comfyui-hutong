# 工具映射与通道（v1 真名，实测）

**先读这一页再动手**：所有工具名以本机 `hb_get_capabilities` + 工具注册表为准。
设计手册里的工具名是「理想契约」，**照它猜会直接打空**。

## 一、三条通道

1. **`hb_*` MCP（首选）** —— 注册表**间歇掉线**，原生直呼名报 `invalid tool call` / 反复 describe
   循环时 → 先 `tool_describe`，再用 **`tool_call(name=..., arguments=...)` 包装调用**。
   只 describe 不 call 会原地打转（这是稳定通道）。
2. **桥命令直投** —— `POST /hermes_bridge/command {type,data,origin}` → 轮询
   `/command/{cid}/result?timeout=`；前端约 1.2s 拉一条命令。
3. **原生 REST** —— `/object_info` `/prompt` `/history` `/queue` `/interrupt` `/view` `/system_stats`。
   绕开画布直接跑图、查节点 schema、看队列都走这里。

**读走后端直出、写依赖前端 JS 轮询** —— 这是判断故障的第一分叉：

| 现象 | 含义 | 处理 |
|---|---|---|
| `/hermes_bridge/ping`、`/graph` 读正常，但 create/update/connect 报「命令超时（前端扩展未响应）」 | 前端 JS 没在响应（浏览器画布页没开 / 页面没注入 JS） | 先确认用户在浏览器开着 ComfyUI；确认不了就改走原生 `/prompt` 提交整图 |
| `hb_connect` 本身失败 | 后端没起来或插件没加载 | 查 `/system_stats`、`custom_nodes/` |

## 二、v1 实有工具（28 个，按用途分组）

| 用途 | 工具 |
|---|---|
| 连接 / 只读 | `hb_connect` `hb_get_state` `hb_get_graph` `hb_get_capabilities` `hb_get_queue` |
| 查资源 | `hb_list_models` `hb_list_workflows` `hb_search_nodes` `hb_list_custom_nodes` `hb_search_custom_nodes` `hb_get_api_prompt` |
| 写画布 | `hb_create_node` `hb_update_node` `hb_move_node` `hb_delete_node` `hb_connect_nodes` `hb_disconnect_nodes` `hb_batch_ops` |
| 校验 | `hb_validate_operations`（投递前校验一批 op）`hb_validate_workflow`（校验当前画布：类型/必需输入/环/输出节点/模型存在） |
| 存档 / 执行 | `hb_save_workflow` `hb_load_workflow`（**破坏性，整张覆盖活画布**）`hb_execute_workflow` `hb_cancel_execution` |
| 安装 | `hb_install_custom_node` `hb_update_custom_node` `hb_uninstall_custom_node` `hb_install_model` |

`hb_get_capabilities` 里 **`restart_comfyui: false`** —— 重启只能让用户手动做（或我 kill 进程重拉）。

## 三、设计手册点名、但 v1 **没有**的能力 → 降级写法

| 手册要的 | 现实 | 降级写法 |
|---|---|---|
| `hb_initialize` | 无 | `hb_connect` + `hb_get_state` |
| `hb_plan_operations` + `hb_apply_operations`（plan_id 两阶段 / 真 dry-run） | 无。只有 `hb_validate_operations`（纯校验，返回值不能被 apply 引用）+ `hb_batch_ops`（直接写） | **无原子事务、无 dry-run**：先 `validate_operations` 过一遍 → 再 `batch_ops` 写 → 读回核对 |
| `hb_open_workflow` | `hb_load_workflow`（破坏性覆盖） | 载入前先存一份当前画布 |
| `hb_search_install_catalog` | `hb_search_custom_nodes` | 同名用途，走本地 Manager 清单 |
| `hb_download_model` / `hb_get_install_job` | `hb_install_model`（Manager 白名单校验） | **无 job 轮询工具**：装完以目标目录/文件存在性验收，别等回执 |
| `expected_revision` / revision 冲突检测 | 桥只给 **graph hash，没有 revision 号** | 改动前记 hash → 写完比 hash；不一致就重读，别硬覆盖 |

**结论**：凡是要写「先 plan 再 apply」「版本冲突重规划」的地方，一律改写成
`validate_operations → batch_ops → 读回 hash/widgets 比对`。写 ComfyUI 相关技能或方案时，
先按这张表校正，别照抄手册。

## 四、常用直查（REST，比 MCP 少一层）

```
GET  /system_stats                 # 就绪判据；含 comfyui_version / vram_free / frontend 版本
GET  /object_info                  # 全部节点注册表（本机注册的全部节点类）；/object_info/<类型> 单查
GET  /queue                        # queue_running / queue_pending
GET  /history/<prompt_id>          # status.completed / outputs.images
POST /prompt    {"prompt":<API格式>, "client_id":"hermes"}
POST /interrupt
GET  /hermes_bridge/ping           # {"protocol":"hermes-comfyui","ok":true,"queued":0}
GET  /hermes_bridge/graph          # 前端最近回报的活画布（nodes+links+槽位登记），无 revision
```

- `/ping`（原生）404 是**正常**的，别用它判死活；用 `/system_stats`。
- 画布进度条 / 连线预览只有**前端自己的 clientId**（`queue_prompt`）才有；外部 client_id 直投
  `/prompt` 只有任务列表、**无连线预览**。

## 五、调用形态坑

- `hb_create_node` / `hb_connect_nodes` 的 `node_id` 传**整型**（字符串报
  `Input validation error: '5' is not of type 'integer'`）；但 `/graph` 快照回读的 node id 是**字符串**；
  `hb_update_node` 两种都见过成功——**调用前看该工具 schema，比较时按各自实际类型处理**。
- `hb_execute_workflow` 可能在 300s 超时但后台继续跑 → 超时后去 `hb_get_queue` + 后端日志确认，
  **别重复提交**。
- 回执不可信：命令可能已执行但回执永远 `COMMAND_TIMEOUT`（实测 create_node 节点已建、
  save_workflow 文件已写，回执全丢）→ 以 graph 快照 / 存档文件为准。
