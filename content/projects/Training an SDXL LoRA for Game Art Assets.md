---
title: "Training an SDXL LoRA for Game Art Assets"
draft: false
created: 2026-11-09 00:04
modified: 2026-11-09 00:24
tags:
  - image-generation
  - sdxl
  - lora
  - comfyui
  - game-art
---

我想测试一个具体的问题：给 SDXL 一小批画风一致的游戏道具图标，能否通过 LoRA，让它把同样的绘制方式应用到新的文字提示上？

在 RTX 3060 Ti 8GB 上，我用 32 张公开图标训练了一组 SDXL LoRA，保存第 40 步和第 80 步的权重，再固定 Prompt 与 Seed，与基础模型做对照。结果很直观：**输出逐渐接近扁平图标风格，但一些原本正确的材质、部件和文字要求也丢失了。**

代码、工作流及全部图片放在 [sdxl-game-icon-lab](https://github.com/John-Ferrel/sdxl-game-icon-lab)。下面记录数据准备、训练配置、实际运行成本和生成结果。

![[sdxl-style-lora-public-comparison.png]]

*先看同一个 Seed 下的两种物品：每行固定 Prompt 和 Seed，三列依次是 SDXL Base、40-step LoRA、80-step LoRA。后面保留了两个 Seed 的全部结果。*

## 1. 实验范围

我使用 SDXL Base 1.0，不训练整个模型，只在 UNet 的注意力投影层上训练 LoRA。文字编码器与 VAE 冻结。目标是 **Style LoRA**，重点观察绘制语言的迁移，以及物体属性能否保留。

这次只研究单件游戏道具图标。为了让结果能够公开复现，我重新选取 [Kenney Generic Items](https://kenney.nl/assets/generic-items) 的 CC0 素材，独立跑了一轮实验。早期使用的私有授权素材、图片和权重没有混入公开结果。

实验配置如下：

| 项目 | 设置 |
|---|---|
| GPU | NVIDIA RTX 3060 Ti，8GB VRAM |
| 基础模型 | SDXL Base 1.0，固定文件 SHA-256 |
| 数据 | 32 张训练图，8 张留出参考 |
| 训练尺寸 | 512 × 512 |
| LoRA | rank 8，alpha 8；UNet `to_q` / `to_k` / `to_v` / `to_out.0` |
| 训练 | 80 次 optimizer update；batch 1，梯度累积 4 |
| 对照 | Base / step 40 / step 80；2 Prompts × 2 Seeds |

8 张留出图片仅作参考。这轮使用训练中出现过的物品类别，比较两个 Prompt、两个 Seed 下的风格、材质和构图变化。

## 2. 数据：让 Caption 只说图片里有什么

素材原本是透明背景的小图标。预处理时，我裁出有效内容、保留长宽比，将最长边放大到 384 像素，再居中合成到浅灰色的 512 × 512 画布上。

![[sdxl-style-lora-dataset-train.png]]

*32 张训练素材，经裁切、缩放和背景合成后统一为 512 × 512。*

![[sdxl-style-lora-dataset-holdout.png]]

*8 张留出参考图。*

每张图有一段短 Caption，例如：

```text
iconstyle, a blue teapot, on a light gray background
```

`iconstyle` 是这次约定的触发字符串，在训练 Caption 和生成 Prompt 中保持一致。

我没有把游戏道具的价格、伤害或稀有度写进 Caption：这些信息通常无法从图像直接观察。把不可见的游戏属性混入视觉描述，会让训练目标更难解释。

40 张原图、划分、Caption 和来源许可都在仓库中。修改数据或描述后，更新数据指纹并重建缓存。

## 3. 8GB 显存下的训练实现

SDXL 的基础权重与文字编码器、VAE 一起加载，会对 8GB 显存造成压力。我采用了两阶段处理：

1. 先用冻结的 VAE 和 CLIP 编码训练图片及其 Caption，将 latent 和文字条件保存为本地缓存。
2. 退出编码进程。真正训练时，只加载基础 UNet、LoRA 和单张缓存，更新 LoRA 参数。

缓存让冻结编码器在训练阶段无需常驻显存。缓存包含数据、Caption、模型和脚本的内容指纹，降低混用不同实验状态的风险。

本次训练配置：

```json
{
  "resolution": 512,
  "rank": 8,
  "lora_alpha": 8,
  "batch_size": 1,
  "gradient_accumulation": 4,
  "mixed_precision": "fp16",
  "gradient_checkpointing": true,
  "optimizer": "AdamW8bit",
  "learning_rate": 0.0001,
  "max_steps": 80,
  "train_text_encoders": false
}
```

每四张图片累积梯度后，优化器才更新一次。因此 80 次更新对应 320 次图片使用；32 张训练图片各被使用 10 次。这组配置用于第一轮小预算实验。

### 训练恢复也需要验证

我先跑到第 8 步，保存训练状态，再继续到第 80 步。中间发生过一次进程中断：日志记录到第 70 步，最近可恢复的完整 checkpoint 却是第 40 步。于是我从 40 恢复，重新计算 41–70，并继续到 80。

因此要区分两种计数：

- **最终 checkpoint 的 optimizer step 为 80**；
- **机器实际执行过 110 次更新**，其中 30 次属于中断后的重算。

训练用的 `.resume.pt` 保存优化器、步数、随机状态等信息；推理使用的 `.safetensors` 只需适配器权重。恢复训练时加载 `.resume.pt`，在 ComfyUI 中生成图片时加载 `.safetensors`。

### 实测资源消耗

| 指标 | 记录值 | 口径 |
|---|---:|---|
| PyTorch allocated 峰值 | 5.24 GiB | 不包括所有非 PyTorch 显存开销 |
| 整卡采样峰值 | 7,749 MiB | 包含桌面等占用；采样可能漏掉瞬时峰值 |
| 正常结束的训练进程合计 | 331.31 秒 | 不包括中断段已消耗的时间 |
| 计入中断段的已知计算时间下界 | 675.50 秒 | 中断时刻未知，因此只是下界 |

本机在 8GB 显存下完成了训练，包含已记录中断段的计算时间至少为 675.50 秒。

## 4. LoRA 学到了什么？

对照时固定 Prompt、Seed、尺寸、采样器和 CFG：512 × 512，Euler / normal，25 inference steps，CFG 7；LoRA 的 model strength 为 1，CLIP strength 为 0。这次**没有加入 ControlNet**，只观察 LoRA 本身带来的差异。

两条 Prompt 分别要求：

```text
iconstyle, a single brass compass with a blue glass center, on a light gray background

iconstyle, a single blue canvas backpack with a top handle and one front pocket, on a light gray background
```

### 示例 A：指南针，风格变简单，材质信息也消失了

固定 `seed=20261026`，下面三张是实际导出的原图，可以分别打开查看边缘和局部细节。

| SDXL Base | LoRA step 40 | LoRA step 80 |
|---|---|---|
| ![[sdxl-style-lora-PUBLIC-compass-base_seed20261026_00001_.png]] | ![[sdxl-style-lora-PUBLIC-compass-40_seed20261026_00001_.png]] | ![[sdxl-style-lora-PUBLIC-compass-80_seed20261026_00001_.png]] |

Base 生成了有金属边框与蓝色中心的装饰性指南针。40 步保留了较多物体细节；80 步则更像简化的深蓝色扁平标识，但 Prompt 要求的黄铜和玻璃中心显著弱化。

### 示例 B：背包，轮廓更扁平，前袋却变得不明显

同样固定 `seed=20261026`：

| SDXL Base | LoRA step 40 | LoRA step 80 |
|---|---|---|
| ![[sdxl-style-lora-PUBLIC-backpack-base_seed20261026_00001_.png]] | ![[sdxl-style-lora-PUBLIC-backpack-40_seed20261026_00001_.png]] | ![[sdxl-style-lora-PUBLIC-backpack-80_seed20261026_00001_.png]] |

80 步的背包更像一个色块清晰的图标，然而布料质感、前袋轮廓以及一些原本正确的形状特征也被简化了。

### 示例 C：换一个 Seed，单件构图仍可能失败

固定 `seed=20261027`，要求仍然是“一只背包”，但三种条件下都出现了近似商品多视图的布局：

| SDXL Base | LoRA step 40 | LoRA step 80 |
|---|---|---|
| ![[sdxl-style-lora-PUBLIC-backpack-base_seed20261027_00001_.png]] | ![[sdxl-style-lora-PUBLIC-backpack-40_seed20261027_00001_.png]] | ![[sdxl-style-lora-PUBLIC-backpack-80_seed20261027_00001_.png]] |

这一 Seed 下，Base 和两个 LoRA checkpoint 都生成了多视图构图。

### 完整对照：两类物品、两个 Seed

![[sdxl-style-lora-all-comparisons.png]]

*四行分别是指南针/背包 × seed 20261026/20261027；每行从左至右是 Base、40 步、80 步。上文放大了其中三行，剩余指南针 Seed 也保留在这张总览里。*

这组对照中，80 步的输出更扁平、色块更清晰，指南针的黄铜材质和背包的布料、前袋细节也随之减弱。背包在第二个 Seed 下仍然出现多视图。下一轮重点检查这些细节在不同 LoRA 强度下如何变化。

### 额外的尺寸检查：768 × 512

除了上述 12 张 512 × 512 的配对对照，我还用第 80 步 LoRA 额外生成了一张 768 × 512 的指南针：

![[sdxl-style-lora-PUBLIC-wide_seed20261026_00001_.png]]

*768 × 512 的独立生成结果：图片成功生成，边缘略软，方向字符出现伪文字。*

## 5. 下一轮怎么调整

这轮先训练到 80 步，保存了 40、80 两个阶段的对照。接下来把风格、材质细节和构图分别记录，与训练 loss 一起观察。loss 对应训练时的噪声预测误差，生成结果则通过固定 Prompt 和 Seed 比较。

下一轮先扩充测试题，再逐项改变变量：

1. 增加物品类别与 Seed，保留所有失败样本。
2. 分别记录风格接近程度、Prompt 遵循和几何结构问题。
3. 对比不同 checkpoint 与 LoRA strength，检查训练阶段和推理强度的影响。
4. 再检查数据覆盖、Caption 与训练长度是否造成差异。
5. 风格效果稳定以后，再加入 ControlNet，单独评估结构控制与风格迁移的组合。

## 6. 如何复现

项目提供了 Windows + NVIDIA 的本地流程、ComfyUI 工作流、固定依赖版本与公开数据。仓库不包含 SDXL 基础权重、LoRA 权重或机器缓存；需要先按 [SETUP](https://github.com/John-Ferrel/sdxl-game-icon-lab/blob/310436b77118d8faa65430e896c02c410054062b/docs/SETUP.md) 配置模型路径和环境。

在仓库根目录、已配置好 Python 环境后，核心过程为：

```powershell
.\.venv\Scripts\python.exe -B scripts/prepare_data.py
.\.venv\Scripts\python.exe -B scripts/cache.py
.\.venv\Scripts\python.exe -B scripts/train.py --stop-after 8 --run-name smoke
.\.venv\Scripts\python.exe -B scripts/train.py --resume runs/public-demo/checkpoints/PUBLIC-ICON-R8_seed20261028_step0008.resume.pt --run-name full
.\.venv\Scripts\python.exe -B scripts/infer.py --suite
```

先运行 Base 工作流，无须训练 LoRA；完成训练后再加载 LoRA 工作流。对于命令、模型下载大小、兼容的 ComfyUI 版本和目录配置，直接参考 [README](https://github.com/John-Ferrel/sdxl-game-icon-lab) 与 SETUP 文档。

训练与生成已在现有 Windows 环境中完成，尚未在另一台干净机器上从零复现。依赖安装、模型路径与 ComfyUI 工作流仍需按实际环境检查。

---

代码与实验：[sdxl-game-icon-lab](https://github.com/John-Ferrel/sdxl-game-icon-lab) · [完整实验记录](https://github.com/John-Ferrel/sdxl-game-icon-lab/blob/310436b77118d8faa65430e896c02c410054062b/results/README.md) · [参数摘要](https://github.com/John-Ferrel/sdxl-game-icon-lab/blob/310436b77118d8faa65430e896c02c410054062b/results/summary.json)


