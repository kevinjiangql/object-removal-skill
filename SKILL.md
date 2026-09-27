---
name: object-removal
description: 本地免费移除照片中的物体、路人、文字、水印、瑕疵，使用 IOPaint (LaMa 等开源模型) inpainting 自动补全背景。当用户要去掉照片里的路人、杂物、水印、文字、电线，或要求物体消除、修图补背景时使用。不依赖云端 API，不用于纯图片生成。
metadata:
  author: eachlabs
  version: "3.1-local"
---

# Object Removal（本地 IOPaint 方案）

用本地开源模型把照片中不需要的元素（路人、杂物、文字、水印、电线、瑕疵）抹掉，并自动用周围背景补全。**完全免费、离线、无 API key**。

核心工具：[IOPaint](https://github.com/Sanster/IOPaint)（原 lama-cleaner），默认模型 **LaMa**，对风景/墙面/地面等重复纹理背景的消除效果最好。

## 环境安装（一次性）

```bash
pip install iopaint
```

要求 Python 3.10+。首次运行会自动下载 LaMa 模型权重（约 200MB）。有 NVIDIA 显卡会自动用 GPU；纯 CPU 也能跑，单张约 10–60 秒。

## 两种工作模式

### 模式 A：Web UI 交互消除（用户在场，最省事）

适合：用户在旁边、想自己框选/点击要消除的对象。

```bash
iopaint start --model=lama --port=8080
```

然后引导用户打开 http://localhost:8080 ：

1. 拖入照片
2. 用画笔涂抹要消除的区域（画笔要**完全盖住**目标，含影子和倒影，稍微涂大一点）
3. 也可启用内置 **Segment Anything 插件**（InteractiveSeg），点击目标自动生成精确 mask，比手涂更准
4. 点消除，满意后下载

大面积物体（超过画面 20–30%）分多次消除，一次一小块，效果比一次涂一大片干净。

### 模式 B：命令行批量消除（已有 mask）

适合：已经有 mask 图，或批量处理。

```bash
iopaint run --model=lama \
  --image=./input_dir \
  --mask=./mask_dir \
  --output=./output_dir
```

- `--image` / `--mask` 可以是单文件或目录
- mask 是**同尺寸黑白图：白色 = 要消除，黑色 = 保留**
- mask 文件名需与图片名一一对应
- `--device=cuda` 或 `--device=cpu` 可手动指定
- **手机照片必须先转正**：iopaint 不读 EXIF 方向标签。竖拍照片常以横置像素存储 + EXIF 旋转标记，直接喂给 iopaint 会让 mask 贴错 90°。先用 `ImageOps.exif_transpose()` 转正另存为 PNG，再对转正图跑 iopaint

## 自然语言消除（"把左边的路人去掉"）

IOPaint 本身不读自然语言。当用户用自然语言指定对象时，按此流程：

1. **自动生成 mask**：用 Grounded-SAM / Grounded-SAM2（GroundingDINO 检测 + SAM 分割），把"路人""垃圾桶""水印"等词转成 mask 图
2. **mask 膨胀**：对 mask 边缘膨胀 10–20 像素（OpenCV `dilate`），确保影子和边缘残留被盖住
3. **mask 校验**：把 mask 轮廓画在原图上并放大检查——mask 边缘距画面主体（要保留的人物）至少 10–20 像素，且完整盖住目标及其影子、倒影
4. **调用模式 B**：`iopaint run --model=lama` 完成消除

如果用户机器上没装 Grounded-SAM，不要现场折腾安装——退到模式 A，让用户在 Web UI 里点选，30 秒搞定。

## 模型选择

| 模型 | 用法 | 适用 |
|------|------|------|
| `lama` | 默认 | 绝大多数消除场景，首选 |
| `mat` | `--model=mat` | 大面积缺失、边缘要求高 |
| `powerpaint-v2` | `--model=powerpaint-v2` | 不只是消除，还想在原位置**重绘新内容**（带 prompt） |
| `sd1.5` 系列 | `--model=runwayml/stable-diffusion-inpainting` | 需要文本引导重绘；吃显存，慎用于纯消除 |

纯消除一律先试 `lama`，效果不好再换 `mat`。

## 输出与验收

1. 输出图与原图同尺寸、同格式
2. 消除后放大检查：纹理重复感、直线断裂（地平线/建筑边缘）、影子残留
3. 有残留 → 把残留区域再涂一次 mask 做二次消除，而不是换模型硬重跑

## 大面积填充失真（模糊/色块）的解决

LaMa 适合小到中等面积（< 画面 15%）；面积过大时上下文不足，会出现模糊、虚化、重复纹理。LaMa 和 cv2 对大面积都不理想（cv2 会出现对角线接缝伪影，LaMa 会糊），且**把大 mask 拆成多块逐块消除是无效的**——未消除的部分会作为上下文污染填充结果。按以下优先级处理：

### 实战验证的两段式工作流（推荐，先做这个）

实测（2026-09 两批手机照片验证）：LaMa 粗消速度极快、远景小点干净，但对 200–600px 的中大型人物常留下**竖向平滑糊柱**，脚部/阴影边界有深色残块。正确做法是两段式，而不是换大模型硬跑全图：

1. **LaMa 粗消**：按模式 B 跑一遍全图消除（CPU 秒级）
2. **放大检查**：把结果按原消除区逐个放大，找出糊柱、深色残块、纹理断裂，记录矩形坐标
3. **SD 只修糊区**：每个糊区裁一个带上下文的小窗送 SD inpainting，修好后羽化贴回；其余像素原样保留

已验证的修复函数：

```python
import cv2, numpy as np, torch
from diffusers import StableDiffusionInpaintPipeline
from PIL import Image

pipe = StableDiffusionInpaintPipeline.from_pretrained(
    "runwayml/stable-diffusion-inpainting", torch_dtype=torch.float32
).to("cpu")
NEG = "people, person, face, text, watermark, buildings"

def sd_region(im, rects, crop, prompt, steps=20, guidance=7.5, feather=6):
    """im: BGR ndarray; rects: 糊区矩形列表（全图坐标）; crop: 裁剪窗 (x1,y1,x2,y2) 全图坐标"""
    x1, y1, x2, y2 = crop
    c = im[y1:y2, x1:x2]
    ch, cw = c.shape[:2]
    m = np.zeros((ch, cw), np.uint8)
    for (mx1, my1, mx2, my2) in rects:
        rx1, ry1 = max(0, mx1 - x1), max(0, my1 - y1)
        rx2, ry2 = min(cw, mx2 - x1), min(ch, my2 - y1)
        if rx2 > rx1 and ry2 > ry1:
            m[ry1:ry2, rx1:rx2] = 255
    s = 512.0 / max(ch, cw)                      # 裁剪窗最长边缩到 512（对齐 8 的倍数）
    tw = max(64, int(round(cw * s / 8)) * 8)
    th = max(64, int(round(ch * s / 8)) * 8)
    c_r = cv2.resize(c, (tw, th), interpolation=cv2.INTER_AREA)
    m_r = cv2.resize(m, (tw, th), interpolation=cv2.INTER_NEAREST)
    res = pipe(prompt=prompt, negative_prompt=NEG,
               image=Image.fromarray(cv2.cvtColor(c_r, cv2.COLOR_BGR2RGB)),
               mask_image=Image.fromarray(m_r), height=th, width=tw,
               num_inference_steps=steps, guidance_scale=guidance).images[0]
    fill = cv2.cvtColor(np.array(res), cv2.COLOR_RGB2BGR)
    fill = cv2.resize(fill, (cw, ch), interpolation=cv2.INTER_CUBIC)
    a = cv2.GaussianBlur(m.astype(np.float32) / 255.0, (0, 0), feather)[..., None]
    out = im.copy()                              # 只把 mask 区域羽化贴回，其余不动
    reg = c.astype(np.float32) * (1 - a) + fill.astype(np.float32) * a
    out[y1:y2, x1:x2] = np.clip(reg, 0, 255).astype(np.uint8)
    return out
```

要点（每条都来自实测踩坑）：
- **必须裁剪送 SD，绝不能跑全图**：SD 输出只有 512px，全图跑会把高分辨率原图毁掉。裁剪窗在 mask 外扩 100–150px 提供上下文
- **prompt 描述该区域应该是什么**（如 "blurred green grassy hillside with scattered dark green shrubs, soft bokeh"），一个区域一个 prompt，按实际背景内容定制
- **只合成 mask 区域**：SD 对未遮挡区的重绘直接丢弃，用高斯羽化（sigma 6）做 alpha 合成避免接缝；diffusers mask 约定为**白 = 填充**
- **跑之前先备份** LaMa 粗消版（`shutil.copyfile`），修完对比验收，不满意可回退

### 方案 1：换 `mat` 模型（大孔专用，同速，首选）

`mat` 论文专门针对大面积缺失训练，对天空、墙面、地面等重复纹理背景的大孔填充明显比 LaMa 干净，**速度和 LaMa 相当**，无需 prompt。

```bash
iopaint run --model=mat --image=photo.jpg --mask=mask.png --output=out
```

首次运行下载约 500MB 模型到 `~/.cache/torch/hub/checkpoints/Places_512_FullData_G.pth`。若 GitHub 下载慢，用 curl 断点续传：
```bash
curl -L --retry 50 --retry-all-errors -C - -o ~/.cache/torch/hub/checkpoints/Places_512_FullData_G.pth \
  https://github.com/Sanster/models/releases/download/add_mat/Places_512_FullData_G.pth
```

### 方案 2：SD inpainting 修糊区（质量最高，两段式的第二段）

实测对比：对 22% 面积的大孔，LaMa 和 MAT 都会模糊，而 SD inpainting + prompt 能自然延续草地、岩石、灌木等纹理，几乎无消除痕迹。**SD 只用于修复 LaMa 糊区，绝不能跑全图**（SD 输出只有 512px，会把高分辨率原图毁掉）——按上文「实战验证的两段式工作流」裁剪小窗逐区修复，完整可复制的 `sd_region()` 代码见该节。

注意：`iopaint run` 不支持 `--prompt` 参数，需直接用 Python + diffusers 加载 pipeline（加载代码见两段式工作流一节）。CPU 单区（512px 窗、20 步）约 3–5 分钟，多区串行处理。

**首次运行**从 HuggingFace 下载约 4.9GB 模型到 `~/.cache/huggingface/hub/`。国内设 `HF_ENDPOINT=https://hf-mirror.com` 加速。下载断连后重跑会自动续传（跳过已缓存文件）；Hub 连接超时会自动回退本地缓存，不影响已装模型使用。

- **prompt 写法**：描述消除后该区域应该呈现的内容（如 "empty wooden floor, natural light"），不要写 "remove the person"；negative prompt 固定 "people, person, face, text, watermark, buildings"
- **步数**：CPU 用 20 步（guidance 7.5）实测够用；有 GPU 用 30–50 步
- **更高质量**：换 `diffusers/stable-diffusion-xl-1.0-inpainting-0.1`（更慢、质量更好）
- **参考图引导**（不想写 prompt）：`Fantasy-Studio/Paint-by-Example`，传一张参考图而非文字

### 方案 3：mask 尽量收紧（配合上述模型）

无论用哪个模型，mask 都只覆盖**必须消除的区域**，不要多涂。mask 越大越容易失真。边缘残留可对 mask 做小幅膨胀（`cv2.dilate` 核 5–10px）后重跑一次。

---

## 常见问题

- **Windows 安装失败**：先 `pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121`（有显卡）或 CPU 版，再装 iopaint
- **大面积消除模糊/色块**：不要拆 mask 分块（无效），换 `mat` 模型；需要生成新内容用扩散模型 + prompt
- **下载模型慢**：`big-lama.pt` 等权重托管在 GitHub Releases，连接不稳时用 `curl -L --retry 50 -C -` 手动下到 `~/.cache/torch/hub/checkpoints/`
- **扩散模型显存不足**：加 `--sd-cpu-textencoder` 或降低 `--sd-num-steps`
- **mask 贴错位置 / 旋转 90°**：手机竖拍 JPG 以横置像素存储 + EXIF 方向标签，而 iopaint 不读 EXIF。先 `ImageOps.exif_transpose()` 转正另存 PNG，对转正图跑 iopaint，并断言输出尺寸与转正图一致
