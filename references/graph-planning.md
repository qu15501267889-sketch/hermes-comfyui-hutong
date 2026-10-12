# 图规划：从需求到最小 DAG

## 一、反向设计四问

1. **产物**：图像 / 视频 / 音频 / 中间结果？以哪个输出节点收尾（SaveImage / PreviewImage / SaveVideo）？
2. **输入**：纯文字 / 已有图像视频音频 / mask / 参考图 / 条件图 / 现有模型？
3. **核心变换**：新生成 / 改图 / 局部重绘 / 插帧 / 放大 / 风格控制 / 合成 / 链式处理？
4. **约束**：尺寸、帧数/时长、模型偏好、速度 vs 质量、seed 可复现、用户现有图是否必须保留？

未指定且不影响主结构的项 → 用本机已有资源 + 安全默认值，**并一句话说明假设**，不要把设计工作推回给用户。

## 二、最小 DAG

- 先定**输出端**，再反推所需输入类型与处理阶段。
- 每个节点必须能说出职责；说不出的**先不加**。
- 同一目的一条明确路径即可。用户没要的脸修复 / 放大 / 多级采样 / 细节增强 / LoRA / ControlNet
  **不自动追加**。
- 分叉只在用户要多个输出/对照/变体时建；合并点检查类型兼容。
- 子流程用语义清晰的组框 + 标题，让画布可读，**不改动未请求的处理逻辑**。

## 三、运行时事实优先

- 节点类型、输入输出 schema、COMBO 合法枚举、允许范围、模型清单——**一律取自当前运行实例**。
- 用真实注册的 **class type**，别把节点显示标题当 class type。
- 连接按 **输入/输出 name + 数据类型**配对；只有序号时才回实时 schema 映射 index。
- 按 name 查不到就 `hb_search_nodes` / `/object_info/<类型>`，**不猜、不拿网页文档代替本机安装状态**。
- 模型引用必须是本机清单里的真实文件；多候选时给简短候选让用户选，别挑一个"看着眼熟"的。
- **前端-only 节点**（`Note` 等）不在 object_info，但在 `LiteGraph.registered_node_types` 里，
  `create_node` 照样能建 —— **object_info 没有 ≠ 建不了**。

## 四、建图铁律（全部实测踩过）

1. **输入槽 index 只能从画布快照按 name 取**。用 object_info 的 required 键序推 = 错位 →
   connect 静默 `CONNECT_FAILED`（FaceDetailer 的 bbox/positive/negative 三条线因此从建起就没连上，
   直到导 api_prompt 才暴露）。
   - **⚠️ autogrow 输入会让后面的槽整体右移（2026-10-12 实测）**：`TextEncodeQwenImage21.images`
     是 `COMFY_AUTOGROW_V3` 动态输入，**一旦连上 `images.image_1`，前端立刻插入一个 `images.image_2`
     空槽** → 其后的 vae / prompt / negative_prompt / resolution 各 +1 位。此时若按连接前读到的下标继续连
     后面几条线，线会全部打偏（实测：负词串到了 prompt 上，vae 直接连空）。
     **两条对策，都要做**：① **把 autogrow 输入放到最后连**（先连 clip/vae/prompt/negative，最后连 images）；
     ② 每次 connect 前**重读快照 + 按目标输入名解析 index**，连完再重试校验。
   - **⚠️ 快照同步 2.5s，连完立刻读会误判"失败"（同日实测）**：连完一条线马上读 `/graph`
     仍显示 `link=None` → 误以为连失败而重连/乱改。**一律用「重试校验」循环**：读→若该槽 link 已是
     非 None 就跳过，否则发一次 connect → 等 2.5s 再读，最多 3 轮。别拿单次读下结论。
2. **widget 名必须先查 `object_info/<类型>`**。`hb_update_node {widgets:{错名:v}}` →
   **静默跳过且返回 ok:true**。经典例子：`ImageScaleToMaxDimension` 的边长 widget 叫
   **`largest_size`**（不是 `max_dimension` / `size`）。改完回读**存档 JSON** 的 `widgets_values`
   逐字段精确核对（`== ["lanczos", 1280]` 这种），别只核 title —— **title 变了不代表值变了**。
   `/graph` 快照的 `widgets_values` 常为 null/过时（读的是前端序列化缓存），`save_workflow` 落盘的才是执行真值。
3. **provider 类常多输出口**：`UltralyticsDetectorProvider` out0=BBOX_DETECTOR / out1=SEGM_DETECTOR
   （连错口 = 类型不匹配）；`SegmDetectorCombined_v2` 直接出 MASK（无需 SEGS 中转）。
4. **类型不同必加转换节点**：MASK→IMAGE 用 `MaskToImage`（SolidMask 直连
   `ImageComposite.destination` 会 CONNECT_FAILED）；4 通道→3 通道用 `SplitImageWithAlpha`。
5. **COMBO 值先从 object_info 查合法枚举再写**（如 FaceDetailer 的 `sam_detection_hint` 合法是
   `center-1` 系）——自造值 = 执行期 `value_not_in_list` 整段拒跑。
6. **connect 成功判据 = 目标 `inputs[slot].link` 已登记**（1.52 fork 的 connect 返回值不可信，
   不查登记就会出现「success 但 links=0」的假成功）。建完用 **api_prompt 反查全链**收口。
7. **改节点类型 ≠ `update_node` 的 type**：`update_node {type:"VAEEncode"}` 返回 True 但**不改
   class_type**（只改 title/widget）。换类型只能**删旧 + create 新 + 重连全部输入输出**；删节点会
   连带拆掉它的所有输入链接（下游 `latent_image` 立刻 link=None），重建后必须逐条重连并核对登记。
8. **离线手写 `widgets_values` 要数全「实际成为 widget 的字段」** = required 里非 input 字段 +
   optional 里的 widget 字段 + seed 类 INT 的 `control_after_generate`。少写一项 → **整条数组错位**。
   错位签名 = 一串值互相串位（`tile_height:'padded'`、`denoise 512.0 bigger than max of 1.0`、
   `cfg:'euler' could not convert string to float` 同轮齐报）→ 改单个值修不好，必须**按 widget 名逐个 set**。
9. **前端 Load + 保存会重排槽序**：离线写入的 `inputs` 序（如 switch,on_false,on_true）会被前端按类
   定义重建（on_false,on_true,switch）——**改线一律按 `inputs[].name` 定位槽**，勿信当初写入的索引。
10. **大批量构造脚本「先落盘再执行」**：一律先 `write_file` 写完整 `.py`（函数/变量全定义、无 print
    引用未定义符、结尾无占位符），`python -m py_compile` 过了再跑。在 execute_code 文本框里边写长构造
    边渐进提交，必被 NameError/SyntaxError 反复打断（曾连续废 4 轮）。
11. **解析「官方推荐采样参数」走 Comfy-Org/workflow_templates**：`api.github.com/repos/
    Comfy-Org/workflow_templates/contents/templates` 列目录 → raw 拉 JSON → 参数常藏在
    `definitions.subgraphs[].nodes`（顶层只有 Load/Save 外壳，找 KSampler 会误判"模板没有采样节点"）。
    **diffusers README 的步数（如 40 步 FlowMatchEuler）是它自家管线标定，与 ComfyUI 值不可互搬**。

## 五、模块开关与透传（核心机制）

### mode 的语义

| mode | 名称 | 行为 |
|---|---|---|
| 0 | Always | 正常执行 |
| 2 | Mute | 从执行图剔除，**不重接** → 下游 required missing → 400 断链。**别用来做开关** |
| 4 | Never | 剔除该节点**并把下游重接到它的第一个节点输入**（1:1 直通，按类型映射验过）|

- **0.37 的 muted 重接只做一跳**：多级 muted 链追不到底 → 下游仍断。
- 远程关：`hb_update_node {node_id, mode:4}`；手动：组内代表节点右键 Never。
- 每组指定一个代表节点（采样器/详情器），组框标题写明「开关 = 节点 X」。

### 采样段整组透传 = 加「出口桥」

**纯图进图出段（1:1 IMAGE）关组会自动透传**；但**采样段组**（`VAEEncodeForInpaint` → KSampler →
VAEDecode）不行：VAEDecode 只有一个 LATENT 输入、没有 IMAGE 可回溯 → 关组直接丢
`61.images / 72.source` → `prompt_outputs_failed_validation` 400。

**标准修法：每组 +1 个 1:1「出口桥」**

```
组入口图 ─→[ImageComposite destination]──(自身 mask=全1 白，1:1 直通)
段输出   ─→[source]                        └→ 下游接下桥输出
```

- 开组：source = 段产物，dest 被全遮 → 桥输出 = 段产物（1:1）。
- 关组：把「段全部节点 **+ 出口桥**」一起 Never → 桥只留 `dest=组入口图`，一跳回组上游 = 整组透传。
- **出口桥必须纳入该组开关名单**（面板映射 / 批量 Never 都要带上），否则只关段不关桥 → 递推进不了组。
- **备选开关 = 改线**（disconnect + connect 换上游）：100% 可靠，适合「一个输入槽挂两条互斥线」
  （量化双线、双底模切换：断 A 连 B，把非活跃线整段 mode=4）。

### A/B 双路互斥

`ComfySwitchNode`（on_false/on_true + switch）只**择一输出**，**未选路照样在执行图里跑**。
真正互斥要把未用整路 `mode=4` 剔除；开关名单 = 整条支路（含上游 loader/编码/解码/缩放）。

### `check_lazy_status` 定论（源码级）

- **机制真实**：`comfy_extras/nodes_logic.py` 的 `SwitchNode.check_lazy_status` —— switch=false 且对侧
  输入为"已连未评估" → 只追加评估选中分支，未选分支上游永不执行。
- **⚠️ 错误判据（已废弃）**：`graphToPrompt` **永远全量** —— 已关分支的节点照常出现在 prompt 键集合，
  这是正常行为，**不是 lazy 失效证据**。禁止用"弃用支路是否从 api_prompt 消失"做判据。
- **生效硬条件**：**被保护分支不能有直连输出节点的旁路**。ComfyUI 执行由输出节点（Preview/Save）
  反向驱动 —— 若每功能 `Preview` 直连 `DEC`，依赖链绕过 Switch → 开关 false 照样拉起整组采样
  （实测：功能已关、GPU 仍卡它 25 步）。**修法 = 各功能预览改接各自 Switch 输出**。
- **正确判据 = 跑图时被关泳道不执行 + 总耗时缩短**（或预览图与原图同秒出）。

### 零 JS 开关架构（替代 JS 面板）

- 每功能尾一个 `ComfySwitchNode`：`on_false`=上游图（透传）、`on_true`=本功能解码，`switch` ← `PrimitiveBoolean`。
- 连锁透传：本开关输出 → 下一功能 `TE.image_1` **和**下一开关 `on_false` → 连关 N 个也一跳到底。
- `PrimitiveStringMultiline → TE.prompt`（widget 转 input）= 纯画布提示词入口；
  全局负词 1 个广播全部 `TE.negative`；`PrimitiveInt` 广播分辨率与 seed —— **改一个数全链变**。
- **`HermesSwitchPanel` / `HermesRetouchPanel` / `HermesIntensityPanel` / `HermesReplacePanel` 是 JS 壳**，
  禁 JS 场景下 = 空壳，别放进新画布（纯 UI 节点，`RETURN_TYPES=()` 不进执行链）。
- **widget 被连线接管后本地输入框变灰 = 设计行为**（值由连线源控制），不是 bug：
  「灰=正常，去控制塔那个数字改，一个数全链同步」。

## 六、增量编辑现有图

- 「当前工作流」= 首要目标，以**活动 graph 快照**为事实，**不靠最近文件名猜身份**。
- 定位用户要改的局部子图，识别进出该子图的边界 links。
- 保留未提到的节点/连线/参数/布局；**不用新模板整图覆盖**。
- 先一句话说清变化范围（如「在当前 decode 后加一条放大支路」），不必描述没改的节点。
- 旧图不完整时报告缺失的关键路径；不必为所有 warning 阻止可独立完成的改动。
- **用户在画布上误删/改坏节点**：先查磁盘存档是否完好（节点数 + 特征类型）→ 完好则
  `hb_load_workflow` 把活画布刷回文件版（零重建）；文件也坏了才重建。

## 七、布局（为读图，不为排版）

- 主数据流**自左向右**；同阶段控制节点放在主流上方/附近。
- 多路条件源放在对应融合/采样节点之前，避免线跨整张画布。
- 阶段间留足空间；相关节点靠近但不重叠；对照/可选路径单独成组。
- **不修改用户已有布局风格**，除非用户让整理版面。
- 大图布局**循环生成坐标，别手拍**（竖排蛇形：每段一行，奇数行左→右、偶数行右→左，上下行端点正对接）。
- 旧画布只当**序列化模板**抄格式（`inputs` / `widgets_values` / `properties`），**拓扑必须重新设计**
  （用户原话「不要照抄 你应该有更好的想法」）。
- id 分区一眼定位：1xx 控制 / 2xx 输入 / 3xx 参考 / 4x0=功能 N / 5xx 输出。

## 八、完成标准

计划里每个必需输出都能沿**类型相容**的连线从输入到达；必需输入都接好或有有效 widget；
用到的模型/节点本机可用；没有意外删除/替换既有内容；读回图后关键 nodes/links 与计划一致。
