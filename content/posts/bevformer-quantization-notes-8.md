+++
date = '2026-09-04T17:00:22+08:00'
title = 'Bevformer Quantization Notes 8'
tags = ['Quantization']
toc = true
+++

# BEVFormer量化笔记8

## 1. 现状

升级 PyTorch 到 2.0.1 了，解决了全部常量折叠问题。之后把 BEVFormer（3 相机、900 queries、15 类）在 Orin/TensorRT 8.5 上以自定义 plugin 部署。导出前对模型做了一组算子替换，降低了模型推理耗时；部署后 `cls_scores` 相对 PyTorch 真值出现明显偏差（`bbox` 基本无偏差）。

## 2. 产生差异的原因

### 2.1. 可能产生差异的链路：

```text
原始 PyTorch fp32  ──(patch/静态化)──▶ patched eager fp32
patched eager fp32 ──(torch.onnx)────▶ ONNX 图
ONNX ─────────────(TRT fp32)────────▶ TRT fp32 engine
TRT fp32 ────────(fp16)─────────────▶ TRT fp16 engine
```

### 2.2 FP32 和 FP16 精度对比

| precision         | bevformer_tiny_fp32_700126_042353 | bevformer_tiny_fp16_260904_103737 |
| ----------------- | --------------------------------- | --------------------------------- |
| bbox_preds_cosine | 0.999920                          | 0.999638                          |
| cls_scores_cosine | 0.687888                          | 0.568498                          |
| bbox_preds_diff   | 13.939384/0.053555/0.000000       | 15.451988/0.160394/0.000000       |
| cls_scores_diff   | 0.627663/0.000056/0.000000        | 0.645458/0.000139/0.000000        |

可以看到 cls_scores_cosine 在 FP32 时，已经下降了。说明 FP16 并不是导致精度下降的真正原因，FP16 只是放大了误差，精度下降在 FP32 已经出现。真正的差异在于原始 PyTorch FP32 和 TensorRT FP32 + plugin。

同时 bbox_preds_cosine = 0.999920 说明：

- 输入基本正确
- query 顺序基本成功
- backbone / BEV / decoder 主体基本正确

问题高度集中在：

```text
decoder hidden state
    -> cls_branch
    -> sigmoid
    -> cls_scores
```

而 mean_diff 很小，但 cosine 只有 0.68。分类输出 900 x 15 = 13500 个元素，而大多数 sigmoid 分数接近 0。

当前结果：

```text
max_diff  = 0.627663
mean_diff = 0.000056
```

说明：

- 大多数位置几乎一致；
- 只有少数 query/class 的分类分数差异很大；
- 这些少数高响应位置会显著拉低 cosine；
- 可能是高分 query 的类别值变化，或者 query 排序/匹配发生局部变化。

因此 cls cosine = 0.688 不代表全部分类输出都错。

当前最有可能原因：

1. TRT plugin 与原始 mmcv MSDA 存在小的 FP32 差异

之前 plugin 单测结果：

```text
SCA FP32 max_abs = 8.2e-6
TSA FP32 max_abs = 1.2e-6
decoder FP32 max_abs = 4.8e-7
```

差异很小，但是会经过：

```text
LayerNorm
FFN
decoder attention
classification linear
sigmoid
```

最终可能在少数分类输出上被放大。

2. SCA 静态化语义可能与原始 SCA 不完全一致

当前 engine 使用：

```text
静态 SCA：每个 query × 每个 camera
```

原始模型使用：

```text
bev_mask.nonzero()
queries_rebatch
Scatter/索引回填
```

理论上两者应等价，但只要无效 camera/query 的处理、mask、slots 归一化存在一个细小差异，就可能影响 decoder hidden state。

`bbox 分支对这种差异可能不敏感，cls 分支则更敏感。

### 2.3. 对模型做的替换

| 原模块 / 算子                                                    | 问题                               | 替换成                                                           | 手段       |
| :--------------------------------------------------------------- | :--------------------------------- | :--------------------------------------------------------------- | :--------- |
| `BEVFormerEncoder.point_sampling`                                | 动态分支 / NaN 处理                | 静态投影 `point_sampling_onnx`（浮点 mask、clamp）               | 静态化     |
| `SpatialCrossAttention.forward`                                  | per-cam nonzero + scatter 动态回填 | 静态 SCA：每 query×每相机 + 浮点 mask 加权                       | 静态化     |
| `MSDeformableAttention3D.forward`（SCA 内 deformable_attention） | mmcv MSDA 无法逐节点展开           | 一个 `MultiScaleDeformableAttnTRT` 节点                          | 自定义节点 |
| `TemporalSelfAttention.forward`                                  | 历史帧折叠引入动态 Slice           | 队列折入 batch、history 常量索引 → `MultiScaleDeformableAttnTRT` | 自定义节点 |
| `CustomMSDeformableAttention.forward`（decoder cross-attn）      | 同上                               | `MultiScaleDeformableAttnTRT` 节点                               | 自定义节点 |
| `mmcv MultiheadAttention`（decoder self-attn）                   | packed in_proj 导出成 257 越界     | 常量切 q/k/v + 头折进 batch → `QKVTRT` 节点                      | 自定义节点 |

cls 的偏差可能来自以上任意一层，设计实验，逐 patch 实验。

## 3. Patch 实验

同一 val 帧、同一 checkpoint：

- 参照（original）：**不替换任何算子的原始模型**（mmcv CUDA MSDA、scatter SCA）FP32；
- 被测（variant）：**只开启指定 subset 的替换**（其余保持原始）FP32；

两者都在同一 GPU eager 跑，比较 `cls_scores` / `bbox_preds` 的 cosine 与 diff。逐层关闭某个替换即可定位是哪一项语义变化导致掉点。

| `--patches`    | 单独开启的替换                     | 能定位                        |
| :------------- | :--------------------------------- | :---------------------------- |
| `sca`          | 静态 `point_sampling` + 静态 `SCA` | SCA 可见性/归一化语义         |
| `tsa`          | 新 `TSA forward`（含内部 `msda`）  | 历史帧合并/placeholder        |
| `msda_encoder` | `encoder MSDA → fused`             | SCA 内 deformable kernel 数值 |
| `msda_decoder` | `decoder MSDA → fused`             | decoder 交叉注意力数值        |
| `mha`          | `decoder` 自注意力 → `QKVTRT`      | 自注意力前投影拆分            |
| `all`          | 全部（基线）                       | 总 drift                      |

结果：

| patches        | cls_cosine | bbox_cosine | cls_max/mean diff   |
| :------------- | :--------- | :---------- | :------------------ |
| original     | 1.000000   | 1.000000    | 0 / 0               |
| all          | 0.828565   | 0.999915    | 0.627761 / 0.000042 |
| sca          | 0.828565   | 0.999915    | 0.627761 / 0.000042 |
| tsa          | 1.000000   | 1.000000    | 0.000001 / 0.000000 |
| msda_encoder | 1.000000   | 1.000000    | 0.000002 / 0.000000 |
| msda_decoder | 1.000000   | 1.000000    | 0.000001 / 0.000000 |
| mha          | 1.000000   | 1.000000    | 0.000000 / 0.000000 |

全部 diff 来自 sca，sca 包含两样东西 `point_sampling_onnx` + 静态 `_sca_forward`。继续排查：

| patches               | cls_cosine | bbox_cosine | cls_max/mean diff   |
| :-------------------- | :--------- | :---------- | :------------------ |
| original            | 1.000000   | 1.000000    | 0 / 0               |
| `point_sampling_onnx` | 1.000000   | 1.000000    | 0.000000 / 0.000000 |
| 静态`_sca_forward`    | 0.828565   | 0.999915    | 0.627761 / 0.000042 |

说明 sca 的掉点完全是由静态 `_sca_forward` 引起的。TSA / MSDA × 2 / MHA 完全无关，所以 bbox 完全不受影响。

## 4. 修复

### 4.1. 修复过程

**结论**：`cls` 精度下降的唯一来源是静态 SCA 对被 mask 的 anchor 做了“图像中心替身”，与原始 `scatter` 的“越界坐标 clamp 到边界”语义不一致；去掉中心替身后 FP32 对比回到 cosine 1.0000（0 diff），FP16 下 cls 0.906 / bbox 0.9997，剩余偏差为纯 FP16 量化。

**现象**：从“原始 PyTorch vs 部署路径（patch eager / TRT engine）”的对比看，`bbox_preds` 几乎无差异（cosine≈0.9999），但 `cls_scores` 明显下降（0.83 量级）；逐 patch 开关定位到唯一来源是 `SpatialCrossAttention` 的静态化。

**分析**：静态 `_sca_forwrad` 与原版有个差异，集中在对不可见 anchor 的处理。

- 原版：每个相机只对“该 query 至少一个 z-anchor 可见”的 query 做 msda，query 带着它全部 D 个 anchor 的原始坐标去采样；落在图外的 anchor 在 mmcv/CUDA 里按采样坐标 `clamp` 到图像边界参与，`softmax` 仍覆盖这些点。
- 静态：对不可见 anchor 一律把坐标换成 0.5（图像中心）再采样，同样参与 softmax。

于是凡是“query 在某个相机部分 anchor 在图外/被 mask”的 BEV 格子，静态版采的是中心、原版采的是边界/真实坐标——两者在该 cell 的特征值不同。相当于把被 mask 的 anchor 坐标换成中心。这类格子数量不多，但恰好在物体边界/遮挡处，经 decoder 后只让少数 object query 的 `cls` 置信度被推过边界（所以 bbox≈0.9999、`cls` 只有 ~0.83、mean diff 极小但 max diff≈0.63）。与原始 `scatter`（采样时把越界坐标 `clamp` 到图像边界）语义不一致。

**修复**：去掉中心，让被 mask 的 anchor 保留原始坐标、由 msda 采样 `clamp` 到边界（与原版语义一致）→ 对比回到 cosine 1.0000、0 diff。

测试新版本静态 `_sca_forward_2`：

| patches                   | cls_cosine | bbox_cosine | cls_max/mean diff   |
| :------------------------ | :--------- | :---------- | :------------------ |
| original（vs original） | 1.000000   | 1.000000    | 0 / 0               |
| 静态`_sca_forward`        | 0.828565   | 0.999915    | 0.627761 / 0.000042 |
| 静态`_sca_forward_2`      | 1.000000   | 1.000000    | 0 / 0               |

### 4.2. 修复后指标

| precision         | bevformer_tiny_fp32_700127_020219 | bevformer_tiny_fp16_700127_004532 |
| :---------------- | :-------------------------------- | :-------------------------------- |
| bbox_preds_cosine | 1.000000                          | 0.999718                          |
| cls_scores_cosine | 0.999995                          | 0.905823                          |
| bbox_preds_diff   | 0.259556 / 0.002006 / 0.000000    | 15.352356 / 0.121248 / 0.000000   |
| cls_scores_diff   | 0.004818 / 0.000001 / 0.000000    | 0.587151 / 0.000104 / 0.000000    |

FP32 已基本无损，FP16 剩余偏差是纯 FP16 量化。
