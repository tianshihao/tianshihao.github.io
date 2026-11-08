+++
date = '2026-08-25T17:41:34+08:00'
title = 'BEVFormer Quantization Notes 3'
tags = ['Quantization']
series = ['BEVFormer量化']
toc = true
+++

# BEVFormer量化笔记3

> 本篇接续笔记 2：Export Engine 流程（第 5 节）与导出问题排查（第 6 节）。

## 5. Export Engine

使用`trtexec`导出BEVFormer，里面经历5个阶段：

1. ONNX解析
1. 图优化
1. 精度标注
1. 内核自动调优
1. 序列化

### 5.1. ONNX解析

TensorRT逐节点读取ONNX计算图，构建内部IR（中间表示）。这一步纯CPU。

### 5.2. 图优化

这是TensorRT的核心加速手段：

- 层融合：`Conv->BN->ReLu`三个kernel合并成一个`CBR` fused kernel。原来要3次显存读写+3次kernel launch，融合后只需要一次。典型ResNet块能减少60%-70%的kernel调用。
- 常量折叠：把可以在构建期算出的常量提前算好。
- 冗余消除：删除推理中不用的输出分支。

### 5.3. 精度标注

TensorRT 根据用户传入的 flag（`--fp16` / `--int8` / `--tf32`）给每一层打上精度标签。量化（PTQ/QAT）在这一步决定哪些层跑 FP16、哪些层跑 INT8，敏感层会回退到高精度。打标签不是最终结果，还要等 5.4 自动调优时结合 kernel 支持度再微调。

### 5.4. 内核自动调优

TensorRT 为每一层在候选的 CUDA kernel（tactic）里挑最快的那个。它会把 tactic 跑一遍真实计时，所以这一步受 `--optShapes` 影响很大——opt 给成 4 而实际跑 1，选出来的 kernel 往往对 B=1 不是最优。细节见 6.3。

### 5.5. 序列化

把选好的图结构、kernel、权重打包成 `.engine` 文件（也称 plan），部署时用 `createInferRuntime` 直接加载，不用再走一遍 5.1–5.4。

## 6. 导出Engine时遇到的问题

### 6.1. ONNX中残留INT64，TensorRT不兼容，被迫截断

Warning 1： INT64权重警告

```text
[08/26/2026-10:22:16] [TRT] [W] onnx2trt_utils.cpp:375: Your ONNX model has been generated with INT64 weights, while TensorRT does not natively support INT64. Attempting to cast down to INT32.
```

Warning 2：INT32截断警告

```text
[01/18/1970-04:54:10] [W] [TRT] onnx2trt_utils.cpp:403: One or more weights outside the range of INT32 was clamped
```

导出的ONNX模型中存在`INT64`类型的权重/常量：

1. `INT64 weights`：ONNX 的 initializer（常量权重）和 Constant 节点中包含了 INT64 类型数据
1. `One or more weights outside the range of INT32 was clamped`：某些 INT64 值超出了 INT32 范围，被强行截断

在我写的`onnx_exporter.py`里面，`INT64`主要有两个来源：

- `manual_bilinear_sample` 中的 `.long()` 调用：`x0 = x.floor().long()` 等会生成 `INT64` 索引张量，这些在 tracing 时会被固化到 ONNX 图中。
- `do_constant_folding=True`：PyTorch 在常量折叠时，会将整数常量以 `INT64` 类型存入 ONNX 的 initializer 中。
  - 这个在Pytorch 2.0.1可以正常工作。

解决方法：

1. `onnx_exporter.py` 里把能确定不超过 INT32 范围的索引/常量，显式 `.int()` / `int32` 而不是 `.long()`，从源头消掉 INT64。
2. 实在消不掉的（比如某些 gather 的索引），在导出后把 INT64 initializer 手动转 INT32，或让 TensorRT 自动 cast（现在的警告就是它自动做的）。

结论：这类 warning 通常无害，前提是**所有被截断的值本身就在 INT32 范围内**（坐标、索引一般都满足）。要警惕的是真正大于 2^31 的值——一旦被 clamp 成 2^31-1，数值就错了。

### 6.2. 动态维度未指定Shape

```text
[08/25/2026-12:03:02] [I] Finish parsing network model
[08/25/2026-12:03:02] [W] Dynamic dimensions required for input: img, but no shapes were provided. Automatically overriding shape to: 1x3x3x544x960
terminate called after throwing an instance of 'std::runtime_error'
  what():  Cannot open model/orin/bevformer_tiny_syzk_detr_dab_anchor_orin_fp32_timing.cache.lock!
export_engine.sh: line 80: 2381129 Aborted                 (core dumped) CUDA_VISIBLE_DEVICES=$GPU_ID bash -c "$CMD"
```

这段日志里其实有**两个**问题：

1. `Dynamic dimensions ... overriding shape to: 1x3x3x544x960`：动态维度没给 shape，TensorRT 乱猜了一个 5D 形状。这是**根因**。
2. `Cannot open ..._timing.cache.lock!`：这是**直接触发点**。TensorRT 在自动调优时要写 timing cache，lock 文件所在目录不存在或没权限，`trtexec` 拿不到锁就抛 `std::runtime_error` 退出。

第二个问题通常是结果而不是原因——只要把 shape 配对，并且确保 `--timingCacheFile` 指向的目录存在且有写权限，就不会走到这一步。

解决方法：在`export_engine.sh`的`trtexec`命令中明确指定输入形状。例如：

```bash
trtexec --onnx=your_model.onnx \
    --saveEngine=your_engine.trt \
    --fp16 \
    --minShapes=img:1x6x3x544x960 \
    --optShapes=img:1x6x3x544x960 \
    --maxShapes=img:1x6x3x544x960
```

### 6.3. 动态batch与shape专题

#### 对于BEV模型来说，img shape是否应该动态？

通常只让batch维度动态，其余维度固定。原因如下：


| 维度              | 建议 | 原因                                                                                                     |
| ----------------- | ---- | -------------------------------------------------------------------------------------------------------- |
| batch（第0维）    | 动态 | 部署时可能batch=1,调试时可能batch=2/4，动态batch灵活度高                                                 |
| num_cams（第1维） | 固定 | BEV模型在导出时，`lidar2img`标定矩阵、`reference_points`等都与摄像头数量绑定，换摄像头数量会破坏模型结构 |
| channels（第2维） | 固定 | 永远是3（RGB）                                                                                           |
| H，W（第3、4维）  | 固定 | BEV网格大小、位置编码、`reference_points`的归一化等都与输入分辨率强相关，还分辨率会破坏模型逻辑          |


#### 什么是batch size？

Batch size就是模型一次前向推理（forward）同时处理的样本数量，对应输入张量`img`的第0维。

对BEVFormer来说，「一个样本」不是一张图，而是一辆车的一组环视图像（`num_cams = 6`）。所以：

```text
img: [B, N, C, H, W] = [1, 6, 3, 544, 960]   # batch=1，一次处理1辆车的6路相机
img: [B, N, C, H, W] = [2, 6, 3, 544, 960]   # batch=2，实际是12张图
```

batch size 本质上是在**延迟、吞吐、显存**三者的取舍：

| 指标 | batch 增大时 | 说明                                                             |
| ---- | ------------ | ---------------------------------------------------------------- |
| 延迟 | 变大         | 单次前向要算更多样本，自动驾驶部署最关心这个                     |
| 吞吐 | 变大         | 单位时间处理的样本数上升，GPU利用率提高，kernel launch开销被摊薄 |
| 显存 | 近似线性增长 | 激活值（activation）按 batch 成倍占显存，容易 OOM                |

- **训练时**：batch 越大梯度越稳，所以用大 batch（配合学习率调整）。
- **部署时**：实时感知要求单帧低延迟，通常就是 `batch = 1`；只有离线批量处理（数据集回放、精度评测）才用大 batch。

所以在部署BEVFormer时，把 batch 设成动态只是为了「同一份引擎既能跑 B=1 也能跑 B=2/4」的灵活性，真正在车端跑的仍然是 B=1。

#### 动态batch在TensorRT里怎么配？

1. ONNX 导出时把第0维标为动态：

```python
torch.onnx.export(
    model, dummy_input, "bevformer.onnx",
    input_names=["img"],
    output_names=["output"],
    dynamic_axes={"img": {0: "batch"}, "output": {0: "batch"}},
    do_constant_folding=True,
)
```

1. 构建引擎时必须给出 batch 的范围，否则就会出现 6.2 的报错：

```bash
trtexec --onnx=bevformer.onnx \
    --saveEngine=bevformer.trt \
    --fp16 \
    --minShapes=img:1x6x3x544x960 \
    --optShapes=img:1x6x3x544x960 \
    --maxShapes=img:4x6x3x544x960
```

1. 三个 shape 的含义不一样：

| 参数          | 作用                                                                            |
| ------------- | ------------------------------------------------------------------------------- |
| `--minShapes` | 运行时允许的最小 batch，小于它会直接报错                                        |
| `--optShapes` | **内核自动调优（tactic selection）的基准**，TensorRT 按这个 shape 挑最优 kernel |
| `--maxShapes` | 运行时允许的最大 batch，显存/workspace 按它预留                                 |

注意 `--optShapes` 会直接影响性能：把 `opt` 设成 `4` 而实际只跑 `1`，选出来的 kernel 对 B=1 往往不是最优。既然部署时实际用的就是 B=1，`opt` 也应该设成 1，`max` 按调试需要给个 2 或 4 即可。

1. 如果运行时 batch 是固定的，还可以用 `--shapes` 直接写死：

```bash
    --shapes=img:2x6x3x544x960     # 与 --minShapes/--optShapes/--maxShapes 互斥
```

#### 什么时候干脆不要动态batch？

如果实际部署只需要一个固定 batch，最简单的做法是导出**静态 shape** 的 ONNX（不写 `dynamic_axes`，dummy input 直接用真实的 `1x6x3x544x960`）：

- TensorRT 能直接读到完整的 5D shape，不需要 `--minShapes/--optShapes/--maxShapes`，也就不会出现 6.2 那种被猜成 `1x3x3x544x960` 的崩溃；
- 没有 shape 分支和运行时 shape 校验，kernel 选择更确定，性能通常也更好；
- 代价是失去灵活性：想跑别的 batch 就得重新导一次 ONNX 再构建一次引擎。

实践中常见的折中是：**只让第0维（batch）动态，其余维度全部固定**，并且 `optShapes` 对齐最常见的使用场景（B=1）。

## 7. 导出检查清单

把上面踩过的坑整理成导出前的自查项：

1. **INT64 是否清零**：能 `.int()` 的都 `.int()`，确认 ONNX initializer 里没有 INT64，或所有 INT64 值都在 INT32 范围内。
2. **动态维度是否配齐**：声明了 `dynamic_axes` 就必须给 `--minShapes/--optShapes/--maxShapes`；不需要动态就直接静态导出。
3. **`optShapes` 是否对准实际部署 batch**：部署 B=1 就把 `opt` 设成 1，别为了「以后可能跑 4」把 opt 给成 4。
4. **timing cache 路径是否存在**：`--timingCacheFile` 指向的目录要提前建好，并且有写权限。
5. **其余维度是否真的需要动态**：BEV 模型里 `num_cams/channels/H/W` 几乎都是固定的，别开多余的动态轴。
