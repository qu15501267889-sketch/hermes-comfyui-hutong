# 桥的 node_id 自动分配会把画布做废（2026-10-13 实测）

## 症状

`create_node` **不传 `node_id`** 时，桥自己分配的 id 会**一路数字拼接**：

```
1 → 11 → 111 → 1111 → 11111 → … → 1111111111111111111111 (≈1e22)
```

（`web/hermes_bridge_v4.js` 的 `nextNodeId(graph)`；快照里这些节点被序列化成
**`"1.1111111111111111e+22"`** 这种浮点字符串。）

一旦 id 超过 **JS 安全整数 2^53（≈9e15）**，这些节点就**无法再被寻址**：

| 操作 | 结果 |
|---|---|
| `connect` | **不落**（投递后等 25s，槽位仍 `link=None`） |
| `delete_node` | **不落**（轮询 60s，节点还在） |
| `update_node`（含改 title） | **不落** |
| `create_node` | ✅ 正常（不依赖既有 id） |

于是画布变成"能加节点、不能连线"的死局 —— 本会话 22 节点的服装工作流因此报废。

## 判据（一眼看出中招）

读 `/hermes_bridge/graph`，看节点 id 列表：

```python
sorted(int(n["id"]) for n in g["nodes"])   # 若出现 1e22 量级 = 中招
# 或者直接看快照 key：'1.1111111111111111e+22'
```

正常 ComfyUI 自己建的节点 id 是 1~500 量级。

## 修法

**每次 `create_node` 都显式传小 node_id**（几百以内），绝不依赖自动分配：

```python
{"class_type": "KSampler", "node_id": 111, "position": [x, y], "title": "..."}
```

显式传 id 是可靠的（本会话多次实测：显式 `node_id=301` 建成后 `connect` 立刻成功，
连线表出现 `('1', 0, '301', 0)`）。

**如果已经中招**：只能让用户**新建一个工作流**（File→New / "+"标签）拿回小 id 画布，
然后用显式小 id 重建。旧节点删不掉，别浪费时间试。

## 顺带发现的两个坑（同一会话）

1. **节点会重复创建**：`create_node` 后快照有 2~4s 延迟，若"读不到就重发"会产生重影
   （一次重发 5 轮 → 画布多出 20 个 90xx 重复节点）。**只发一次，然后只轮询**。
2. **`StringConcatenate` 的 `string_a/string_b/delimiter` 是纯 widget，接不上线**
   （快照里它们出现在 `inputs` 数组，但前端 `node.inputs[i]` 不存在 → CONNECT_FAILED）。
   要做字符串拼接请用 **`StringFormat`**：它的 `values` 是 `COMFY_AUTOGROW_V3` 真输入槽，
   配 `f_string="…{a}…{b}…"` 即可连线（同 `TextEncodeQwenImage21.images` 的机制）。
