# Object Removal Skill

本地免费的物体/路人消除 Skill，基于 [IOPaint](https://github.com/Sanster/IOPaint)（原 lama-cleaner）+ Stable Diffusion Inpainting 两段式工作流。**完全本地、离线、无 API key、零成本**。

适用于：去除照片中的路人、杂物、文字、水印、电线、瑕疵，并自动补全背景。

## 特性

- **本地运行**：LaMa 模型仅约 200MB，CPU 秒级出图，无需显卡
- **两段式工作流**：LaMa 粗消 + SD inpainting 精修，兼顾速度与质量
- **实战验证**：在真实手机照片上完成多批人物消除，纹理自然、无接缝
- **多种场景**：Web UI 交互消除 / 命令行批量处理 / 自然语言指定对象

## 安装

```bash
pip install iopaint
```

要求 Python 3.10+。首次运行自动下载 LaMa 权重（约 200MB）；SD inpainting 模型约 4.9GB，仅在需要精修时下载。

## 使用方式

### 模式 A：Web UI 交互消除

```bash
iopaint start --model=lama --port=8080
```

打开 http://localhost:8080 ，拖入照片、涂抹要消除的区域（盖住目标及影子）、点消除。

### 模式 B：命令行批量消除

```bash
iopaint run --model=lama --image=./input_dir --mask=./mask_dir --output=./output_dir
```

mask 为同尺寸黑白图：**白色 = 消除，黑色 = 保留**。

### 自然语言消除

Skill 内置完整流程：自动生成 mask（Grounded-SAM）→ mask 膨胀 → mask 校验（距主体 10–20px）→ 调用模式 B。

## 两段式工作流（核心）

单靠 LaMa 处理 200–600px 的中大型人物会留下竖向平滑糊柱和脚部残块。推荐的流程：

```
LaMa 粗消（CPU 秒级） → 逐区放大检查 → SD inpainting 只修糊区（羽化贴回）
```

关键要点：

1. **SD 必须裁剪小窗送入**，绝不能跑全图（SD 输出仅 512px）；裁剪窗在 mask 外扩 100–150px 提供上下文
2. **prompt 描述该区域应该是什么**（草地/灌木/岩石），而非"要消除什么"；negative prompt 固定为 `people, person, face, text, watermark, buildings`
3. **只合成 mask 区域**：高斯羽化（sigma 6）alpha 混合避免接缝；diffusers mask 约定为白 = 填充
4. 修复前备份 LaMa 粗消版，便于对比回退

完整的已验证 `sd_region()` 修复函数见 [SKILL.md](SKILL.md)。

## 常见坑

| 问题 | 原因与解法 |
|------|-----------|
| mask 贴错位置 / 旋转 90° | 手机竖拍 JPG 以横置像素存储 + EXIF 标签，iopaint 不读 EXIF。先 `ImageOps.exif_transpose()` 转正另存 PNG |
| 大面积消除模糊/色块 | 拆 mask 分块无效（未消除部分污染上下文），改用两段式工作流 |
| 主体被误伤 | mask 边缘距要保留的主体至少 10–20 像素，先画轮廓放大检查 |
| HF 连接超时 | 设 `HF_ENDPOINT=https://hf-mirror.com`；超时会自动回退本地缓存 |

## 文件说明

- `SKILL.md` — Skill 主文件，包含完整的工作流程、已验证代码和踩坑记录
