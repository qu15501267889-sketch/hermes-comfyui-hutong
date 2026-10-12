---
name: hermes-comfyui-hutong
description: 用 ComfyUI 或 HermesBridge 插件操作画布时加载：最少工具调用建/改/验证工作流，不绕远路。
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [comfyui, hermesbridge, mcp, canvas, workflow-automation]
    category: creative
---

# ComfyUI 画布操作（HermesBridge 互通）

只要**用到 ComfyUI 或 `hb_*` 插件**就加载本技能。唯一目标：**用最少的工具调用，把需求变成一张连得对、验证过的节点图**——不绕远路、不返工、不顺手多做。

## When to Use

- 在当前画布新建 / 修改 / 排错节点工作流（图像、视频、音频）
- 用 `hb_*` 工具读画布、加节点、连线、改参数、打组、加开关
- 用户说「搭/改工作流、加个节点、把某段关掉、改个数、跑一张看看」
- 起 ComfyUI、确认插件/节点是否加载、判断画布实况

## Don't use for

- 装/升 ComfyUI 本体、显卡选型 → `comfyui-deployment-ops`
- 修图**方案设计**（磨皮/瘦身/打光怎么规划）→ `comfyui-retouch-workflow`
- 与画布无关的纯出图 API 调用、批量 sweep → `comfyui`
- 只要提示词建议、不读写画布

## 开工：读一次，别反复读

1. `hb_connect`（或 `hb_get_state`）一次 → bridge / 后端 / 队列状态
2. `hb_get_graph` 一次 → 当前活画布（nodes + links + 槽位）
3. **比对活画布 `mode` vs 最近存档 JSON 的 `mode`** → 不一致 = 用户改了没保存，**先汇报再动**，绝不擅自 load/save
4. 上面两样不够时，才定向 `hb_search_nodes` / `hb_list_models`


## 任务分类 → 最短调用序列

| 类型 | 触发语 | 序列 |
|---|---|---|
| EXPLAIN | 解释这张图/这段在干嘛 | connect + get_graph → 作答，**不调写工具** |
| BUILD_NEW | 搭个新的… | connect + get_graph + 定向 schema/模型 + `validate_operations` + `batch_ops` + 读回 |
| MODIFY | 在这加/改一段 | connect + get_graph（只看相关子图）+ 定向 schema + `batch_ops` + 读回 |
| FIX | 报错/断了/参数不对 | 读报错节点与后端日志 → 查该节点槽位与 widget 名 → 最小 update/connect → 只重验这一段 |
| BUILD_AND_RUN | 搭好并跑 | 上面全部 + `validate_workflow` + 执行（渲染要画布可见，见下） |

## 写画布五条铁律

1. **槽位只能按 name 取** —— `/hermes_bridge/graph` 的 `inputs[].name → index`。用 `object_info` 的 required 键序去推 = 错位 → connect 静默失败。
2. **widget 名先查 `object_info/<类型>` 再写** —— `hb_update_node` 遇到错的 widget 名**静默 no-op 却返回 ok**（假成功）。改完回读存档 `widgets_values` 逐字段核对。
3. **一段 DAG 合一次 `hb_batch_ops`** —— 不要「建一个节点 → 重读全图 → 再建下一个」。
4. **类型不匹配必加转换节点** —— MASK→IMAGE = `MaskToImage`；4 通道→3 通道 = `SplitImageWithAlpha`。
5. **关模块用 `mode=4`（Never），不用 `mode=2`（Mute）** —— Mute 剔除但不重接 = 下游 required missing 400。采样段整组透传要配「出口桥」（见 `graph-planning.md`）。

## 不绕远路：禁令表

| 禁 | 正解 |
|---|---|
| 建一个节点→重读全图→再建下一个 | 一段 DAG 一次 `hb_batch_ops` |
| 改完立刻读图（拿到旧数据又改一遍） | 等 3~4s（前端 syncGraph 2.5s 周期）再验 |
| 信 `COMMAND_TIMEOUT` 回执→重发 | 回执不可信，读 graph 快照 / 读存档核对 |
| 桥工具报"不存在"后原地反复 describe | `tool_describe` → `tool_call(name, arguments)` 包装 |
| 画布坏了→整图重建 | 先查磁盘存档是否完好 → `hb_load_workflow` 刷回 |
| 顺手执行 / 顺手装插件下模型 | 构建与执行分离，没让跑就不跑 |
| 每次任务重新列模型/节点清单 | 同一次任务内复用，状态变了才刷 |
| 只输出"接下来我将…"不调工具 | 计划一句话带过，当轮直接调 |

**调用预算**：简单改图 = 1 次连接 + 1 次读图 + 少量定向查（可并行）+ 1 次预校验 + 1 次批量写 + 1 次定向读回。超预算必须由新证据或具体错误触发，不为凑顺序凑次数。

## 构建与执行分离

- 「搭 / 加 / 改 / 布置」= 只建图，**不跑**；「跑一下 / 出图 / 测试」才执行，执行前先 `hb_validate_workflow`。
- 用户在场时渲染要走**画布可见**的通道（前端 `queue_prompt` 或让他点 Queue），别把渲染藏进后台脚本交差。
- 汇报分两句：**画布搭好了没**（按读回说）+ **任务跑了没**（按 `/history` 说）。没有读回不许说"已搭好"。

## 收尾自检（答不上来 = 没完成）

1. 这次几次调用？有没有重复调用？
2. 我核验的是**我改的那部分**吗（不是只看工具返回 `ok`）？
3. 有没有做用户没让做的事（执行 / 安装 / 删图 / 覆盖存档）？

## references（按需读）

- `tool-map.md` —— v1 工具真名映射、三条通道、设计手册点名但**不存在**的能力与降级写法
- `graph-planning.md` —— 反向设计、最小 DAG、槽位/类型/widget 铁律、开关与透传机制、布局
- `workflow-patterns.md` —— 文生图 / 图生图 / 局部重绘 / 放大 / 条件参考 / 视频 的蓝图
- `pitfalls.md` —— 失败分类速查 + 局部恢复步骤 + 回归自检清单
