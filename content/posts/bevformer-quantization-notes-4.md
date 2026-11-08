+++
date = '2026-08-27T16:24:15+08:00'
title = 'BEVFormer Quantization Notes 4'
tags = ['Quantization']
series = ['BEVFormer量化']
toc = true
+++

# BEVFormer量化笔记4

> 本篇接续笔记 3：性能分析（第 7 节）与 layer info 分析（第 8 节）。

## 7. 性能分析

结论：INT8 相比 FP16 没有性能收益，精度却差得多（见 7.2）；FP16 吞吐约为 FP32 的 1.9 倍，是后续改进的方向。

### 7.1. 性能

INT8性能并未提升，且模型本身受Orin资源占用影响，耗时波动大


| performance                      | FP32高负载                 | FP16低负载                 | FP16高负载                 | INT8低负载                 | INT8高负载                 |
| -------------------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------- | -------------------------- |
| Throughput                       | 2.6826 qps                 | 5.0788 qps                 | 2.7949 qps                 | 5.0647 qps                 | 2.6911 qps                 |
| Latency (max/mean/min)           | 382.755/372.773/359.955 ms | 203.397/196.897/190.550 ms | 384.442/357.798/316.573 ms | 202.538/197.446/195.458 ms | 419.296/371.597/324.387 ms |
| Latency median                   | 373.278 ms                 | 198.540 ms                 | 364.538 ms                 | 197.350 ms                 | 375.708 ms                 |
| Latency percentile(90%)          | 379.852 ms                 | 201.924 ms                 | 374.321 ms                 | 199.032 ms                 | 399.520 ms                 |
| Latency percentile(95%)          | 380.251 ms                 | 202.654 ms                 | 375.742 ms                 | 199.412 ms                 | 401.939 ms                 |
| Latency percentile(99%)          | 381.929 ms                 | 203.358 ms                 | 377.053 ms                 | 201.150 ms                 | 416.175 ms                 |
| H2D Latency (max/mean/min)       | 4.605/3.124/2.183 ms       | 3.961/3.167/2.868 ms       | 3.857/2.707/2.222 ms       | 4.959/3.125/2.753 ms       | 5.729/3.133/2.331 ms       |
| H2D Latency median               | 3.029 ms                   | 3.094 ms                   | 2.626 ms                   | 3.026 ms                   | 2.964 ms                   |
| H2D Latency percentile(90%)      | 3.743 ms                   | 3.440 ms                   | 3.195 ms                   | 3.507 ms                   | 4.104 ms                   |
| H2D Latency percentile(95%)      | 4.105 ms                   | 3.587 ms                   | 3.363 ms                   | 3.638 ms                   | 4.646 ms                   |
| H2D Latency percentile(99%)      | 4.197 ms                   | 3.837 ms                   | 3.706 ms                   | 4.707 ms                   | 5.531 ms                   |
| GPU Compute Time (max/mean/min)  | 378.690/369.103/356.528 ms | 199.926/193.127/187.139 ms | 380.242/353.396/313.835 ms | 197.329/193.987/192.298 ms | 410.706/367.622/321.857 ms |
| GPU Compute Time median          | 369.869 ms                 | 194.784 ms                 | 359.157 ms                 | 193.899 ms                 | 371.218 ms                 |
| GPU Compute Time percentile(90%) | 376.354 ms                 | 198.015 ms                 | 371.048 ms                 | 195.617 ms                 | 396.222 ms                 |
| GPU Compute Time percentile(95%) | 377.252 ms                 | 198.908 ms                 | 372.035 ms                 | 196.021 ms                 | 398.074 ms                 |
| GPU Compute Time percentile(99%) | 378.442 ms                 | 199.774 ms                 | 372.940 ms                 | 196.821 ms                 | 408.112 ms                 |
| D2H Latency (max/mean/min)       | 4.944/0.447/0.104 ms       | 1.219/0.505/0.136 ms       | 9.832/1.603/0.107 ms       | 1.159/0.232/0.121 ms       | 7.631/0.743/0.103 ms       |
| D2H Latency median               | 0.434 ms                   | 0.495 ms                   | 0.628 ms                   | 0.166 ms                   | 0.420 ms                   |
| D2H Latency percentile(90%)      | 0.863 ms                   | 0.872 ms                   | 5.198 ms                   | 0.496 ms                   | 1.188 ms                   |
| D2H Latency percentile(95%)      | 1.119 ms                   | 0.903 ms                   | 7.693 ms                   | 0.555 ms                   | 3.149 ms                   |
| D2H Latency percentile(99%)      | 1.440 ms                   | 1.179 ms                   | 9.814 ms                   | 0.782 ms                   | 5.623 ms                   |
| Total Host Walltime              | 37.277 s                   | 19.690 s                   | 35.780 s                   | 19.745 s                   | 37.160 s                   |
| Total GPU Compute Time           | 36.910 s                   | 19.313 s                   | 35.340 s                   | 19.399 s                   | 36.762 s                   |

从表里能读出三件事：

1. **低负载下 FP16 与 INT8 几乎一样快**（吞吐 5.08 vs 5.06 qps，约为 FP32 的 1.9 倍），INT8 没有带来额外收益——与第 8 节「层数膨胀淹没计算加速」的分析一致。
2. **高负载下三种精度都掉到 2.7–2.8 qps**，且延迟 max/min 差距拉大，说明瓶颈在 Orin 资源争用，不在计算精度。
3. **H2D + D2H 中位数合计不到 4 ms**，相对 200/370 ms 量级的 GPU Compute Time 可以忽略，耗时基本全在 GPU 计算上。

### 7.2. 精度

FP16精度需要改进，INT8精度过低。


| precision         | FP32                        | FP16                        | INT8                        |
| ----------------- | --------------------------- | --------------------------- | --------------------------- |
| bbox_preds_cosine | 0.999887                    | 0.999590                    | 0.998596                    |
| cls_scores_cosine | 0.926067                    | 0.831727                    | 0.357765                    |
| bbox_preds_diff   | 15.084946/0.134497/0.000000 | 17.616318/0.241345/0.000000 | 24.715466/0.540364/0.000000 |
| cls_scores_diff   | 0.945730/0.000194/0.000000  | 0.938262/0.000326/0.000000  | 0.978774/0.000642/0.000000  |


## 8. 分析layer info

错了错了，拿到模型之后应该先分析layer info！！！导出的模型layer可能有精度回退……

结论：**FP16和INT8出现了严重了层数膨胀，膨胀源是非计算层的爆炸式增长，完全淹没了精度降低带来的计算加速。**


| 指标                | FP32  | FP16     | INT8       |
| ------------------- | ----- | -------- | ---------- |
| 总层数              | 99    | 488 (×5) | 1753 (×18) |
| 计算层占比          | 82.8% | 23.8%    | 28.1%      |
| Reformat 层         | 5     | 23       | 144        |
| Slice + NoOp + Fill | 7     | 323      | 775        |
| FP32 回退           | 0     | 9        | 0          |


### 8.1. FP16分析

#### 8.1.1. 结构膨胀

FP16新增了389层！

| 类型  | 增量 | 来源                                 |
| ----- | ---- | ------------------------------------ |
| Slice | +154 | 动态shape padding mask / stride 展开 |
| NoOp  | +99  | Reshape / Unsqueeze 显式化           |
| Fill  | +63  | 动态 shape 下的 Linspace 索引生成    |
| 其他  | +73  | ElementWise、Shuffle、Myelin 等      |


#### 8.1.2. FP32回退

```text
Slice_2584 ────[FP16→FP32]───┐
FP16 输入1 ────[FP16→FP32]────┤
FP16 输入2 ────[FP16→FP32]────┤──→ ForeignNode[Add_2568...Unsqueeze_2943]
FP16 输入3 ────[FP16→FP32]────┤      ↑ Myelin 执行，FP32
FP16 输入4 ────[FP16→FP32]────┘
Reshape_25 ──────────────────┘ (NoOp, FP32)
```

一个 Myelin ForeignNode 导致 9 层回退。Myelin 是 TRT 的自动融合引擎，它把 Add + Unsqueeze 融合成了一个 ForeignNode，但这个融合后的算子不支持 FP16，被迫回退到 FP32，并且所有输入需要 FP16→FP32 的 reformat。


### 8.2. INT8分析

#### 8.2.1. 结构膨胀

INT8膨胀到了1753层，FP32只有99层，膨胀了17.7倍

| 层类型          | 数量 | 占比  | 说明                     |
| --------------- | ---- | ----- | ------------------------ |
| NoOp            | 556  | 31.7% | Reshape/Unsqueeze 显式化 |
| PointWiseV2     | 247  | 14.1% | 逐元素操作（Q/DQ 相关）  |
| Constant        | 193  | 11.0% | 量化权重常量             |
| Slice           | 156  | 8.9%  | Padding mask             |
| Reformat        | 144  | 8.2%  | FP32↔INT8 转换           |
| CaskConvolution | 141  | 8.0%  | 实际卷积计算             |
| Fill            | 63   | 3.6%  | 动态索引生成             |
| 其他            | 253  | 14.4% |

#### 8.2.2. INT8 计算层精度分析

虽然 Summary 显示 → FP32: 336 (68.3%)，但 Reformat 详情揭示了真相：

```text
FP32 ──→ Row major Int8 format  (量化)
        ↓
   [INT8 卷积计算]
        ↓
Row major Int8 format ──→ Row major linear FP32  (反量化)
```

INT8 引擎的 Q/DQ 模式导致每个 INT8 计算层前后各有一个 Reformat，这解释了 144 个 Reformat 的来源。计算层本身的输出格式是 FP32（反量化后），所以脚本统计为 FP32，但实际计算确实是 INT8。

#### 8.2.3. INT8 额外问题

- 193 个 Constant：FP32 只有 4 个。INT8 需要把权重从 FP32 量化到 INT8，每个量化权重块都变成了一个独立的 Constant 层
- 247 个 PointWiseV2：FP32 只有 19 个。大量逐元素操作来自量化/反量化链中的 scale/zero_point 运算
- 556 个 NoOp：与 FP16 相同的根源——Reshape/Unsqueeze 显式化，但 INT8 的 Q/DQ 链进一步放大了这个数字


### 8.3. 根因

#### 8.3.1. 动态shape泛滥

证据：

- Slice 从 2 膨胀到 156（所有引擎）
- Fill 从 0 膨胀到 63（所有引擎）
- Tile_302 的 Size: [-25,4,8192,3]，Mode=WRAP，使用隐式 padding mask

原因：ONNX 模型使用了动态维度（如 batch 可变、序列长度可变），TRT 必须为每个动态维插入 Slice/Fill 层来处理 padding 和索引生成。

#### 8.3.2. Reshape/Unsqueeze 过度显式化

证据：

- NoOp 从 5 → 104 → 556
- Unsqueeze 相关的 reformat 和 copy 节点大量出现

原因：ONNX 导出时未做 shape 推理和常量折叠，每个 Reshape/Unsqueeze 都被保留为独立节点。

#### 8.3.3. Myelin 融合回退（FP16 特有）

证据：ForeignNode[Add_2568...Unsqueeze_2943] 不支持 FP16，导致 9 层回退。

原因：Myelin 自动融合了 Add + Unsqueeze，但该融合算子未注册 FP16 kernel。

### 8.4. 修复

#### 8.4.1. 导出ONNX时消除动态Shape

预期效果：Slice 从 156 → ~10，Fill 从 63 → 0，INT8 的 NoOp 减少 30%+

#### 8.4.2. 消除Myelin FP16回退

```bash
# 在 trtexec 构建时，禁用 Myelin 或限制其融合范围
trtexec --onnx=model.onnx \
        --fp16 \
        --tacticSources=-myelin \     # 禁用 Myelin tactic
        --saveEngine=fp16.engine
```

预期效果：消除 9 个 FP32 回退 + 5 个 reformat

**状态**：8.4.1 与 8.4.2 为待执行项；下面 4 条（P1/P2）仅作记录，暂未执行。

🟠 P1 — 重要优化

**3. ONNX 简化（必做）**

```bash
# 安装 onnx-simplifier
pip install onnx-simplifier

# 简化 ONNX 模型（常量折叠 + shape 推理 + 消除冗余节点）
python -m onnxsim model.onnx model_sim.onnx \
    --dynamic-input-shape \
    --input-shape "input:1,3,800,800"
```

预期效果：NoOp 减少 50%+，Reshape/Unsqueeze 被折叠，Slice 减少

**4. 使用 onnx-graphsurgeon 清理冗余节点**

```python
import onnx_graphsurgeon as gs

graph = gs.import_onnx(onnx.load("model_sim.onnx"))

# 消除连续的 NoOp 节点
for node in graph.nodes:
    if node.op == "Identity":
        node.outputs[0].inputs.clear()
        node.inputs[0].outputs = node.outputs
        node.inputs.clear()
        node.outputs.clear()

graph.cleanup()
onnx.save(gs.export_onnx(graph), "model_clean.onnx")
```

🟡 P2 — INT8 量化改进

**5. 使用 PTQ 显式量化（替代 TRT 隐式量化）**

```python
# 使用 TensorRT Model Optimizer 做显式量化
# 这比 TRT 自动 INT8 校准产生的 Q/DQ 节点更少、更可控
from tensorrt_llm.quantization import quantize

# 或使用 PyTorch Quantization
import torch.quantization as quant

model.qconfig = quant.get_default_qconfig('fbgemm')
model = quant.prepare_qat(model, inplace=True)
# ... 训练/校准 ...
model = quant.convert(model, inplace=True)
```

原因：TRT 隐式量化（--int8）会为每个张量独立插入 Q/DQ，导致 144 个 Reformat + 247 个 PointWiseV2。显式量化可以手动控制 Q/DQ 位置，减少 60% 以上的转换节点。

**6. 如果必须用 TRT 隐式 INT8**

```bash
trtexec --onnx=model_sim.onnx \
        --int8 \
        --calib=calibration.cache \
        --layerPrecisions="*:int8" \      # 强制所有层 INT8
        --layerOutputTypes="*:fp32" \     # 统一输出 FP32
        --tacticSources=-myelin \         # 禁用 Myelin
        --saveEngine=int8.engine
```

## 附录

### TensorRT Layer Info Analysis

#### Summary Table


| Metric               | FP32 Engine | FP16 Engine | INT8 Engine   |
| -------------------- | ----------- | ----------- | ------------- |
| Total Layers         | 99          | 488         | 1753          |
| FP32                 | 99 (100.0%) | 9 (3.0%)    | 1051 (100.0%) |
| FP16                 | 0 (0.0%)    | 289 (97.0%) | 0 (0.0%)      |
| N/A                  | 0 (0.0%)    | 190 (0.0%)  | 702 (0.0%)    |
| Tactic Nonzero       | 80 (80.8%)  | 105 (21.5%) | 658 (37.5%)   |
| Tactic Zero          | 19 (19.2%)  | 383 (78.5%) | 1095 (62.5%)  |
| Reformat Layers      | 5 (5.1%)    | 23 (4.7%)   | 144 (8.2%)    |
| Compute-Heavy Layers | 82 (82.8%)  | 116 (23.8%) | 492 (28.1%)   |
| → FP32               | 82 (100.0%) | 1 (0.9%)    | 336 (68.3%)   |
| → FP16               | 0 (0.0%)    | 115 (99.1%) | 0 (0.0%)      |
| FP32 Fallback        | 0           | 9           | 0             |

#### Layer Type Distribution


| LayerType              | FP32 Engine | FP16 Engine | INT8 Engine |
| ---------------------- | ----------- | ----------- | ----------- |
| NoOp                   | 5           | 104         | 556         |
| PointWiseV2            | 19          | 22          | 247         |
| Constant               | 4           | 14          | 193         |
| Slice                  | 2           | 156         | 156         |
| Reformat               | 5           | 23          | 144         |
| CaskConvolution        | 55          | 55          | 141         |
| Reduce                 | 0           | 2           | 80          |
| Fill                   | 0           | 63          | 63          |
| Shuffle                | 0           | 7           | 48          |
| ElementWise            | 0           | 11          | 40          |
| CaskGemmConvolution    | 1           | 5           | 32          |
| Myelin                 | 7           | 22          | 23          |
| CaskGemmMatrixMultiply | 0           | 0           | 11          |
| CublasConvolution      | 0           | 1           | 9           |
| CudaSoftMax            | 0           | 0           | 6           |
| CaskPooling            | 0           | 1           | 0           |
| Scale                  | 0           | 1           | 1           |
| ShapeHostToDevice      | 0           | 1           | 1           |
| CaskConvActPool        | 0           | 0           | 1           |
| CudnnPooling           | 1           | 0           | 0           |
| MatrixMultiply         | 0           | 0           | 1           |

#### Reformat Details: FP32 Engine

Total: 5

- `Unsqueeze_3427_copy_input`: Row major linear FP32 → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 0 to reshape_after_MatMul_3454`: Channel major FP32 format where channel % 4 == 0 → Row major linearFP32
- `4964 copy`: Row major linear FP32 → Row major linear FP32
- `Reformatting CopyNode for Output Tensor 0 to MaxPool_28`: Row major linear FP32 → Channel major FP32 format where channel % 4 == 0
- `Reformatting CopyNode for Input Tensor 2 to {ForeignNode[5239...Slice_4578]}`: Channel major FP32 format where channel % 4 == 0 → Row majorlinear FP32

#### Reformat Details: FP16 Engine

Total: 23

- `Slice_324`: Row major linear FP16 format → Row major linear FP16 format
- `Unsqueeze_800_copy_output`: Row major linear FP16 format → Row major linear FP16 format
- `1645 copy`: Row major linear FP16 format → Row major linear FP16 format
- `Slice_830`: Row major linear FP16 format → Row major linear FP16 format
- `Reformatting CopyNode for Input Tensor 0 to Conv_26 + Relu_27`: Row major linear FP32 → Channel major FP16 format where channel % 4 == 0
- `Reformatting CopyNode for Input Tensor 3 to {ForeignNode[1523...Reshape_1690]}`: Channel major FP16 format where channel % 8 == 0 → Row major linear FP16 format
- `Slice_1707`: Row major linear FP16 format → Row major linear FP16 format
- `Slice_2584`: Row major linear FP16 format → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 0 to {ForeignNode[Add_2568...Unsqueeze_2943]}`: Row major linear FP16 format → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 1 to {ForeignNode[Add_2568...Unsqueeze_2943]}`: Row major linear FP16 format → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 2 to {ForeignNode[Add_2568...Unsqueeze_2943]}`: Row major linear FP16 format → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 3 to {ForeignNode[Add_2568...Unsqueeze_2943]}`: Row major linear FP16 format → Row major linear FP32
- `Unsqueeze_3427_copy_input`: Row major linear FP16 format → Row major linear FP16 format
- `4964 copy`: Row major linear FP16 format → Row major linear FP16 format
- `Unsqueeze_3441_copy_input`: Row major linear FP16 format → Row major linear FP16 format
- `Reformatting CopyNode for Input Tensor 0 to Expand_2957`: Row major linear FP32 → Row major linear FP16 format
- `Reformatting CopyNode for Input Tensor 6 to {ForeignNode[5239...Add_4052]}`: Row major linear FP16 format → Channel major FP16 format wherechannel % 8 == 0
- `Reformatting CopyNode for Input Tensor 7 to {ForeignNode[5239...Add_4052]}`: Row major linear FP16 format → Channel major FP16 format wherechannel % 8 == 0
- `Reformatting CopyNode for Input Tensor 8 to {ForeignNode[5239...Add_4052]}`: Row major linear FP16 format → Channel major FP16 format wherechannel % 8 == 0
- `Reformatting CopyNode for Input Tensor 9 to {ForeignNode[5239...Add_4052]}`: Row major linear FP16 format → Channel major FP16 format wherechannel % 8 == 0
- ... (3 more)

#### Reformat Details: INT8 Engine

Total: 144

- `Slice_324`: Row major linear FP32 → Row major linear FP32
- `Unsqueeze_800_copy_output`: Row major linear FP32 → Row major Int8 format
- `1645 copy`: Row major Int8 format → Row major Int8 format
- `Slice_830`: Row major Int8 format → Row major linear FP32
- `Reformatting CopyNode for Output Tensor 0 to {ForeignNode[model.pts_bbox_head.positional_encoding.col_embed.weight...Add_536]}`: Row major linear FP32 → Row major Int8 format
- `Reformatting CopyNode for Input Tensor 0 to reshape_before_MatMul_850`: Row major linear FP32 → Row major Int8 format
- `reshape_after_MatMul_850_copy_input`: Row major Int8 format → Row major linear FP32
- `reshape_after_MatMul_842_copy_input`: Row major Int8 format → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 0 to Conv_26 + Relu_27 + MaxPool_28`: Row major linear FP32 → Row major Int8 format
- `Reformatting CopyNode for Input Tensor 0 to Transpose_1210 + Reshape_1220 + reshape_before_MatMul_1250`: Row major linear FP32 → Row major Int8 format
- `reshape_after_MatMul_1250_copy_input`: Row major Int8 format → Row major linear FP32
- `Reformatting CopyNode for Input Tensor 0 to Reshape_1230 + reshape_before_MatMul_1266`: Row major linear FP32 → Row major Int8 format
- `reshape_after_MatMul_1266_copy_input`: Row major Int8 format → Row major linear FP32
- `reshape_after_MatMul_1258_copy_input`: Row major Int8 format → Row major linear FP32
- `Reformatting CopyNode for Output Tensor 0 to PWN(PWN(PWN(PWN(PWN(2702 + (Unnamed Layer* 2710) [Shuffle], Add_1645), Sqrt_1646), Div_1647), Mul_1648), Add_1649)`: Row major linear FP32 → Row major Int8 format
- `Reformatting CopyNode for Output Tensor 0 to model.pts_bbox_head.transformer.encoder.layers.0.ffns.0.layers.0.0.bias + (Unnamed Layer* 2724) [Shuffle]`: Row major linear FP32 → Row major Int8 format
- `Reformatting CopyNode for Input Tensor 0 to PWN(Add_1654, Add_1655)`: Row major linear FP32 → Row major Int8 format
- `Reformatting CopyNode for Output Tensor 0 to PWN(Add_1654, Add_1655)`: Row major Int8 format → Row major linear FP32
- `Reformatting CopyNode for Output Tensor 0 to PWN(PWN(PWN(PWN(PWN(2721 + (Unnamed Layer* 2741) [Shuffle], Add_1662), Sqrt_1663), Div_1664), Mul_1665), Add_1666)`: Row major linear FP32 → Row major Int8 format
- `Unsqueeze_1677_copy_output`: Row major Int8 format → Row major Int8 format
- ... (124 more)

#### FP32 Fallback: FP16 Engine

Total fallback layers: 9

- **[Reformat]** x7: `Slice_2584`, `Reformatting CopyNode for Input Tensor 0 to {ForeignNode[Add_2568...Unsqueeze_2943]}`, `Reformatting CopyNode for Input Tensor 1 to {ForeignNode[Add_2568...Unsqueeze_2943]}`, `Reformatting CopyNode for Input Tensor 2 to {ForeignNode[Add_2568...Unsqueeze_2943]}`, `Reformatting CopyNode for Input Tensor 3 to {ForeignNode[Add_2568...Unsqueeze_2943]}` ... (2 more)
- **[NoOp]** x1: `Reshape_25`
- **[Myelin]** x1: `{ForeignNode[Add_2568...Unsqueeze_2943]}`
