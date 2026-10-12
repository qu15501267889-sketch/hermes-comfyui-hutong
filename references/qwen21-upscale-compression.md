# Qwen-Image 分块放大：vae_compression 必须填 16（2026-10-12 实测踩坑）

## 症状（用户原话）

> "一直到光影这一栏都是正常的，然后超分完就变得不仅失真了，而且也没有放大，
> 而且还有很明显的噪点。"

- 放大后**尺寸和放大前完全一样**（992×1280 → 992×1280）
- 细节被抹平：脸部高频残差 std **5.98 → 3.75**（-37%），背景 3.61 → 2.40
- 皮肤变塑料感、背景出现涂抹/糊化（事后视觉复核确认「AI 修复痕迹明显」）

## 根因

`ComfyUI-Impact-Pack/modules/impact/impact_pack.py`：

```python
class IterativeLatentUpscale:            # 行 1521
    def doit(self, samples, upscale_factor, steps, temp_prefix, upscaler,
             step_mode="simple", vae_compression=8, unique_id=None):
        h, w = samples['samples'].shape[-2:]          # latent 尺寸
        w, h = w * vae_compression, h * vae_compression   # ← 换算成"像素尺寸"
        ...
        new_w = w*scale; new_h = h*scale               # 目标尺寸从这里算
        current_latent = upscaler.upscale_shape(step_info, current_latent, new_w, new_h, temp_prefix)
```

`IterativeImageUpscale`（行 1587）先把像素 VAEEncode 成 latent，再调它。

- **Qwen-Image 的 VAE 是 16 倍空间压缩**（8× 下采样 + 2×2 patchify），不是 SD 的 8 倍。
- 节点默认 `vae_compression=8` → 992×1280 的 latent(62×80) 被当成 496×640 的图
  → `upscale_factor=2` 算出目标 992×1280 = **正好等于原尺寸**。
- 更糟：它先把 latent 缩到 ~46×60 再拉回 62×80，**等于用 denoise 0.4 重画两遍却没放大**
  → 纯劣化（这解释了"失真 + 没放大"；用户口中的"噪点"实为过度平滑+涂抹）。

**判据（一眼看出填错了）**：看日志 `IterativeLatentUpscale[1/2]: WxH`：
- 填 8 → 打印 `744.0x960.0`（≈原图一半）
- 填 16 → 打印 `1488.0x1920.0`（=原图 ×1.5）✅
即 **打印值应 ≈ 实际输入像素 × 迭代 scale**。若打印值约为输入的一半，就是 compression 填小了。

## 修法

| 节点 | widget | 正确值 |
|---|---|---|
| `IterativeImageUpscale` (312) | `vae_compression` | **16**（Qwen-Image / Flux 类 16× VAE） |
| 3000 万分之一意外 | | SD1.5/SDXL/SD3 用 8 |

改完 `vae_compression=16`，992×1280 → **2016×2560**（×2）真放大。

## ⚠️ 代价：8G 卡上真放大非常慢

修好之后**才开始真干活**，实测：
- 迭代两轮（`steps=2` + `step_mode=simple`）：1.5× 轮 → 2.0× 轮
- 512 tile / 16 步 / denoise 0.4：进度条 144 步，首轮 ~3.3s/it，全程 **20~30 分钟**
- 日志反复出现 `loaded partially` / `Unloaded partially` → Q8 主模型(~9GB) 超过 8G 可用显存，
  **每步都在换入换出**（显存抖动 = 慢的主因）

**结论（重要）**：8G 卡不适合「Qwen 直接当超分器做 2K 分块精修」。
优先方案：
1. **两段式（推荐）**：`UpscaleModelLoader`(ESRGAN 类, 如 4x-UltraSharp) 秒级完成放大，
   Qwen 只做低 denoise(0.2~0.3) 精修 → 快 10 倍且细节更好；把该 model 接到
   `PixelTiledKSampleUpscalerProvider.upscale_model_opt` 即可。
2. **降档**：倍数 1.5×、`steps`(迭代轮数) 1、采样步数 16→8 → 约 5~8 分钟。
3. 主模型降量化档（Q5/Q4）减少换页。

## 相关坑

- 该链是 Impact Pack 的 `UPSCALER` 类型：`PixelTiledKSampleUpscalerProvider`(出 UPSCALER)
  → `IterativeImageUpscale.upscaler`。Qwen 当超分器就是靠这个 Provider 带 model/positive/negative。
- 放大段同样**必须走 ComfySwitchNode**：`PreviewImage` 若直连放大输出（不走开关），
  它是输出节点 → 引擎为执行它必然把整条放大链拉起来，**开关关了也照样跑**（本会话实测，
  332s 白跑）。四段预览接开关、唯独新增的放大预览漏接 = 踩过。
