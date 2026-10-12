# hermes-comfyui-hutong

**Hermes ⟷ ComfyUI 互通技能包**：让 Hermes 用**最少的工具调用**，把一句需求变成一张连得对、验证过的 ComfyUI 节点图——不绕远路、不返工、不顺手多做。

配套插件：[qu15501267889-sketch/ComfyUI-HermesBridge](https://github.com/qu15501267889-sketch/ComfyUI-HermesBridge)（28 个 `hb_*` MCP 工具，直连真实画布，零浏览器自动化）。

## 这个技能解决什么

| 痛点 | 本技能的做法 |
|---|---|
| 逐节点建→重读全图→再建下一个，来回几十轮 | 一段 DAG 合一次 `hb_batch_ops` |
| 改完立刻读画布，拿到旧数据又改一遍 | 等 3~4s（前端 syncGraph 2.5s 周期）再验 |
| 工具回执超时 → 以为失败 → 重发 | 回执不可信，以 graph 快照 / 存档文件为准 |
| 桥工具偶发"不存在"，反复 describe 打转 | `tool_describe` → `tool_call(name, args)` 包装 |
| 画布被改坏就整图重建 | 先查磁盘存档 → `hb_load_workflow` 刷回 |
| 只让"搭图"却顺手跑了/顺手下了模型 | 构建与执行分离，没让跑就不跑 |
| 改个 widget 名字写错，工具却返回 ok | 先查 `object_info`，改完回读存档逐字段核对 |

## 技能边界（避免和别的 ComfyUI 技能抢触发）

**适用**：在当前画布新建 / 修改 / 排错节点工作流；用 `hb_*` 工具读画布、加节点、连线、改参数、加开关；起 ComfyUI 与判断画布实况。

**不适用**：
- 安装 / 升级 ComfyUI 本体、显卡选型 → 用部署类技能
- 修图**方案设计**（磨皮/瘦身/打光怎么规划）→ 用设计类技能
- 与画布无关的纯出图 API 调用、批量 sweep → 用通用 ComfyUI CLI/API 技能
- 只要提示词建议、不读写画布

## 安装

**方式 A：拷进 Hermes 技能目录**

```
cp -r hermes-comfyui-hutong ~/.hermes/skills/creative/
```

**方式 B：外部目录挂载**（不想拷贝、要跟仓库同步）

在 Hermes 配置里把本仓库的父目录加进 `skills.external_dirs`：

```yaml
skills:
  external_dirs:
    - "<你的路径>/hermes-comfyui-hutong-parent"
```

新开一个会话即可被发现；`hermes skills list` 能看到 `hermes-comfyui-hutong` 即安装成功。

## 前置条件

1. ComfyUI 在跑（默认 `http://127.0.0.1:8188`）。
2. `ComfyUI-HermesBridge` 插件已装进 `custom_nodes/`（提供 `/hermes_bridge/*` 端点与 `hb_*` MCP 工具）。
3. MCP server 已接进 Hermes，工具名以 `hb_` 开头。

任一条不满足时，本技能**不会**假装已经操作了画布——只会报告 blocker。

## 文件结构

```
SKILL.md                      # 主流程：边界 / 开工三步 / 任务分类 / 五条铁律 / 绕远路禁令 / 自检
references/
├── tool-map.md               # v1 工具真名映射、三条通道、设计手册点名但不存在的能力与降级写法
├── graph-planning.md         # 反向设计、最小 DAG、槽位/类型/widget 铁律、开关与透传机制、布局
├── workflow-patterns.md      # 文生图 / 图生图 / 局部重绘 / 放大 / 条件参考 / 视频 的蓝图
└── pitfalls.md               # 失败分类速查 + 局部恢复 + 资源安装下载坑 + 回归自检清单
```

`references/` 按需加载：只在对应场景才读，主文件保持短。

## 触发

只要用到 ComfyUI 或 `hb_*` 插件就会被加载——包括"搭个工作流""在这加个节点""把这段关掉"
"改个数""跑一张看看""起一下 ComfyUI"。

## 设计原则（都来自实测踩坑）

- **运行时事实优先**：节点类型、槽位、COMBO 枚举、模型清单一律取自运行实例，不猜、不照抄网页文档。
- **增量优先**：改局部，保留未提到的节点/连线/参数/布局，不用新模板整图覆盖。
- **写后读回**：只有读回核验过才说"已搭好"；成功只按读回说，失败要定位到具体节点/槽位。
- **调用预算**：简单改图 = 1 连接 + 1 读图 + 少量定向查 + 1 预校验 + 1 批量写 + 1 读回；
  超预算必须由新证据或具体错误触发。
- **构建与执行分离**：执行、覆盖工作流、安装/下载分别是不同动作，不自动扩张。

## 许可

MIT
