# 失败分类、恢复步骤、回归自检

## 一、按「观察到的现象」查表

| 现象 | 先读什么 | 最小修复 | 一次验证 |
|---|---|---|---|
| 写命令全部「命令超时（前端扩展未响应）」，但读正常 | 用户是否开着 ComfyUI 画布 tab | 让用户确认页面在；或改走原生 `/prompt` 直投跑通后再落画布 | `/graph` 是否变化 |
| 回执超时但怀疑已执行 | `/graph` 快照、磁盘存档文件 | **不要再发一遍** | 节点数/关键 type 与预期一致 |
| 改完读图拿到旧值 | —（时间问题，不是 bug） | 等 3~4s（syncGraph 2.5s 周期）再读 | 第二次读与第一次一致 |
| 「Success」但画布没变 | `inputs[].link` 登记、`widgets_values` 真值 | 命令可能没落地：读登记判据重发 | 目标槽 link 已登记 |
| `status: success` 但整段没跑、2-3 秒秒回、后段不在 outputs | **后端 stderr 日志** `grep -iE "Failed to validate\|Required input is missing"` | 补上缺失的 required 输入 | 重跑，outputs 含后段节点 |
| 输出是常数灰（RGB≈127/std<8） | 用 PIL 读 `stddev` | 全图调整段把 `VAEEncodeForInpaint` 换普通 `VAEEncode` | 重跑读 stddev |
| 4 通道报错 `expected 3 channels, but got 4` | 解码输出通道数 | 超分前插 `SplitImageWithAlpha`（转 RGB） | 重跑 |
| 一串值互相串位（`tile_height:'padded'`、`denoise 512.0 > max 1.0`） | 该节点的 widget 名清单 | **按 widget 名逐个 set**（不要按数组索引） | 回读存档逐位核对 |
| `value_not_in_list` 整段拒跑 | `object_info/<类型>` 的 COMBO 合法枚举 | 写入合法枚举值 | 重跑 validation 通过 |
| 换节点类型没生效 | —（`update_node.type` 不改 class_type） | 删旧 + 建新 + 重连全部输入输出 | 槽登记逐条核对 |
| 进程无故消失 / 自己关了 | Application 日志有无 `Id=1000/1001/1002`；System 日志 NVIDIA 驱动事件 | 无崩溃事件 = 驱动重置（查 System 日志 NVIDIA 驱动事件：驱动重置会把 python 进程无声带走） | 重启后队列 0/0 |
| 图全黑 / 5KB PNG | 后端日志 `invalid value encountered in cast` | 查 GGUF 反量化精度 + 前缀误判回退（见下） | 读像素 stddev，不只看 status |
| 纯颗粒噪声、数值全正常 | conditioning stats 干净 ≠ 语义对 | 先同 prompt **重跑一次**；仍噪查文本编码器 loader 类型 | vision 亲眼看图 |
| 中断后日志刷 `Cancelling running prompt` 死循环、队列不清 | `/queue` running 非 0 不降 | **只能杀进程重启**（`POST /interrupt` 救不回） | 重启后 `/queue` 0/0 |
| 前端 queuePrompt 卡 `QUEUE_BUSY` | —（后端重启后 processingQueue 不复位） | 刷新画布 | 能排队 |
| 载入后回读是**旧版**内容 | `/graph` 节点数 vs 文件节点数 | **再 `load_workflow` 一次**（回执丢了没关系，以状态为准） | 回读与文件一致 |

## 二、状态判定与「哪边是权威」

- **活画布 vs 磁盘存档可以不一致，且差异有信息量**：用户在浏览器改了没保存。
  照存档下结论 = 判错当前状态；照存档 load/save = **抹掉用户未保存的改动**。
  规矩：先读活画布逐节点比对 → 不一致**先汇报并问「接着跑还是开回来」** → 汇报前绝不 load/save。
- 判「哪边更新」= `os.path.getmtime(文件)` vs 你最后一次写入时刻。
- **别靠会话搜索还原画布状态**：直接读磁盘 workflow JSON（节点数/type/title/mode/widgets_values 全在）
  + `/hermes_bridge/graph`（活画布实况）。画布名/中文分段名 FTS 常召回不到；两源冲突以**活画布**为准
  （它可能比文件新），文件只在用户明说「我弄坏了」时才当复原源。
- **链路断言前先读开关/mode 基线**：面板开关部分关闭时，下游"跳级"连到上一个**开着**的模块输出
  = 正确透传，不是错位。期望表必须**按当前开关状态现推**，别套"全开"版。
- **探测 widget 执行真值的「临时存档三步」**：`hb_save_workflow{name:"__tmp_probe"}` →
  读 `workflows/__tmp_probe.json` 的 `widgets_values` → 删文件（回执丢了文件也在）。

## 三、浏览器缓存覆盖（离线改文件的必踩）

ComfyUI 刷新后**从浏览器本地缓存自动恢复上次 graph（≠磁盘文件）**；用户一点保存 = 旧版盖回磁盘，
**离线写入的修复整批蒸发**（曾整批重写 2~3 遍）。
判据 = `os.path.getmtime(文件) > 你上次写入时刻`，或标志节点消失（写入后记节点数 + 标志类型，
下次动手前先核这两样）。

**必胜加载协议（交用户执行）**：
1. 首选**无痕 / InPrivate 窗口**开画布地址（零缓存必读磁盘）→ 从工作流列表点开；
2. 普通窗口则**先关全部 ComfyUI 标签**再开，别理会自动恢复出的画布，直接列表点开；
3. **跑图前肉眼核对标志节点**（本次改动的类型与数量 + 总节点数）—— **核对不过 = 别跑更别保存**；
   核对过 → 跑图 → 效果对了**再**保存。
4. 多开标签 = 每页各跑 JS interval（实测 5 倍频）+ 命令通道互抢 → **只留一个标签**。

## 四、运行/交付通道选择

- 用户在场时**渲染必须在画布可见**（原话校正：「为啥是后台跑？我要看到是节点在跑」）：
  走前端 `queue_prompt`（前端自己的 clientId → 连线预览 + 节点动画）或让他点 Queue。
- 挂 `/history` 轮询的脚本只当**观察员等结果**，回复里一句话说明分工，别把渲染藏进后台脚本交差。
- 验收闭环：`hb_get_api_prompt` 反查 → `POST /prompt`（400 时抓 body 看 `node_errors`）→
  `/history/<id>` 轮询 `status.completed` → `outputs.images` → **文件尺寸/像素核对**。
- **画布 prompt 经原生 `/prompt` 独立复跑 = 定性画布配置正确性的手段**：若原生 API 出正常图、
  而经某交付通道出坏图 → 坏的是**交付通道**，别改画布配置。

## 五、GGUF / 新架构模型加载（通用坑，不限某模型）

- **架构白名单**：GGUF 文件 `general.architecture` 是新名字、而 `ComfyUI-GGUF/loader.py` 的
  `IMG_ARCH_LIST` 没这项 → `ValueError: Unexpected architecture type in GGUF file: '<arch>'`。
- **更隐的坑**：强放白名单但前端误判架构 → **静默当 FLUX 加载**（后端 `[INFO] model_type FLUX`）
  → 结构错配 → NaN 全黑图。判据 = 出图后**读像素**（PIL stddev / 5KB 黑 PNG），别只信 `status: success`。
- **二阶根因（ComfyUI 核心 bug，不限某模型）**：`comfy/model_detection.py` 的
  `unet_prefix_from_state_dict` 只查 `model.diffusion_model.` / `model.model.` / `net.` 三个前缀，
  匹配数 ≤5 就**无脑回退 `"model."`**；而**无前缀顶层** key 的模型会被误判 → 前缀替换把全部 key
  过滤成 0 个 → 架构识别彻底失败 → 回退 FLUX → 全黑。
  诊断脚本：复现 `unet_prefix_from_state_dict` → 前缀替换 → `model_config_from_unet` 三步并打印结果（打印 prefix / temp_sd 数 / model_type）。
  修法 = 在该函数开头对无前缀特征 key 直接 `return ""`，改完**重启**。
- 反量化精度：GGUF 高级加载器的 `dequant_dtype` 过高低会让 latent 溢出 NaN；
  降到 float16 是**画质零差别**的省显存操作（反量化中间缓存精度而已，可逆）。

## 六、资源安装 / 下载（`hb_install_*` 相关，动手前必读）

**Manager 队列 HTTP 契约**（实测，违者 500）：

| 端点 | 硬性要求 | 违反后果 |
|---|---|---|
| `POST /manager/queue/install` | body 必带 `channel:"default"` + `mode:"cache"` | 缺键 KeyError → **500** |
| `POST /manager/queue/uninstall` / `queue/update` | 只要 `{id, ui_id, version:"unknown", files:[url]}`（**不要** channel/mode） | — |
| `POST /manager/queue/install_model` | 必带 `save_path` + `base` + `filename`，与 model-list 条目精确一致 | 缺 `save_path` → 500；对不上 → 400 |
| `GET /customnode/getlist` | query 必带 `?mode=cache`（或 local） | 缺参 → 500 |

- **队列三步固定**：`enqueue(200)` → `POST /manager/queue/start`（**常被漏掉**）→
  `GET /manager/queue/status` 轮询至 `total_count==done_count && !is_processing`。
- **结果不回传 HTTP**（只走 WebSocket/日志）→ **以目标目录/文件存在性验收**。
- **title ≠ 目录名**：git clone 目录取 URL basename（清单 title `"ComfyUI Impact Pack"` → 目录
  `ComfyUI-Impact-Pack`），验证/卸载别用 title 拼路径。
- 清单搜索优先读**本地** `ComfyUI-Manager/custom-node-list.json`（绕开 getlist 的 500）。
- **装 pip 包前必须停 ComfyUI**：运行中 python 进程锁 `cv2.pyd` → `WinError 5 拒绝访问`，整个安装回滚。
- **curl 下载（Windows 原生 curl.exe）**：`-o` 必须给 `D:/...` 形式；**MSYS 路径 `/d/AI/...` → exit 23
  静默写失败**（`$( )` 拼出来的同样不行）。推荐形态：
  `curl --fail -sL -C - -m <秒> -x http://<proxy> -o "D:/.../file" <url>`
  （`--fail` 防 404 存 HTML、`-C -` 断点续传、`-L` 跟重定向）。下完读文件头校验（safetensors 头长度 +
  JSON 数 tensor；`PK\x03\x04` = zip 容器的 .pth，也正常）。
- **后台启动 ComfyUI 时 stdout 与 stderr 都要重定向到文件**：Manager 内部大量 `print`，stdout 接在
  失效管道上 → flush `OSError EINVAL` → **全端点间歇 500**。

## 七、绕远路禁令（详细版，每条都实测踩过）

| 禁 | 正解 |
|---|---|
| 建一个节点→重读全图→再建下一个 | 一段 DAG 合一次 `hb_batch_ops` |
| 改完立刻读图（旧数据→又改一遍） | 等 3~4s（syncGraph 2.5s）再验 |
| 信 `COMMAND_TIMEOUT` / 「命令超时」→ 重发 | 回执不可信，读 graph 快照 / 存档核对 |
| 桥注册表掉线后原地反复 `tool_describe` | `tool_describe` → `tool_call(name, args)` 包装，一步到位 |
| 画布坏了 → 整图重建 | 先查磁盘存档完好 → `hb_load_workflow` 刷回 |
| 顺手执行 / 顺手装插件 / 顺手下模型 | 构建与执行分离，没让跑就不跑 |
| 每次任务重新列模型/节点清单 | 同一次任务内复用，状态变了才刷 |
| 一遇疑点就钻去读插件源码 / 写诊断基建 | **先跑后查**：建好连好设好 → 直接跑 → 看后端日志/看像素 → 报错再针对性查 |
| 改值类请求先搭探针做对照实验 | 读 `object_info` 真 widget 名 → update → 回读存档验证，三步收工（诊断只在用户问"为什么"时做） |
| 只输出"接下来我将…"不调工具就结束回合 | 计划一句话带过，当轮直接调；空转会换来用户连续「继续」催 |
| 改节点**参数**时顺手换节点类型 | 换类型只能删+建+重连 |
| 把 `Note` / 前端-only 节点当成"建不了" | object_info 没有 ≠ 建不了（registry 里有就能建） |

## 八、回归自检清单（改完 SKILL.md / references 后跑一遍）

每次修订本技能后，挑 7 类场景自查，记录「工具调用序列 + 是否偏航」：

| # | 场景 | 期望 |
|---|---|---|
| 1 | 简单改参数（改个 denoise / 边长） | 定向读 schema → update → 回读存档核对。**不搭基建** |
| 2 | 在当前图后加一段（不覆盖既有） | 只读相关子图；batch 写；读回只验新增段 |
| 3 | 用户说「搭个工作流」但没让跑 | **不执行**、不下载模型；只报画布状态 |
| 4 | 桥写操作全超时 | 一次诊断通道（区分读/写）→ 报告 blocker + 下一步，**不连发写请求** |
| 5 | 用户只说「改成视频工作流」而当前图未保存 | 提具体影响 + 在「覆盖」与「另存」间要一次选择，**不静默删图** |
| 6 | 工具返回成功但 graph hash / 节点数没变 | 以读回画布为准，精准诊断该命令，**不把接受命令当落图成功** |
| 7 | 一个节点建好但连线后图没通 | 查 `inputs[].link` 登记 + api_prompt 反查，**不做全图扫描重来** |

**效率指标**（记录基线，不追求绝对越少）：状态初始化次数 / 完整画布读取次数（是否只读一次）/
schema-model 查询是否只与选定方案相关 / 写操作是单批还是逐节点往返 / 有无无新信息的重复调用 /
核验是否覆盖本次变更而非只看 `success`。
